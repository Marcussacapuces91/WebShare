# 🌐 WebShare

WebShare vise à rendre les fichiers d’un ESP32 accessibles depuis un navigateur, y compris lorsque l’appareil fonctionne en **Wi-Fi SoftAP sans connexion Internet**. Le projet privilégie une interface de secours toujours disponible et une interface enrichie lorsqu’elle peut être chargée.

## 🧭 Sommaire

- [🎯 Objectif](#-objectif)
- [🗂️ Organisation de LittleFS](#️-organisation-de-littlefs)
- [🖥️ Interface Web](#️-interface-web)
- [🔌 Backend HTTP et API](#-backend-http-et-api)
- [📤 Téléversements fiables](#-téléversements-fiables)
- [⚡ Performances et concurrence](#-performances-et-concurrence)
- [🔐 SHA-256](#-sha-256)
- [📚 Catalogue des fichiers](#-catalogue-des-fichiers)
- [🕒 Gestion du temps](#-gestion-du-temps)
- [🧑‍💻 Répartition des responsabilités](#-répartition-des-responsabilités)
- [👀 Prévisualisation](#-prévisualisation)
- [🏗️ Architecture cible](#️-architecture-cible)
- [➡️ Prochaine étape](#️-prochaine-étape)

## 🎯 Objectif

Depuis un navigateur Windows, Linux, macOS ou mobile, l’utilisateur doit pouvoir :

- parcourir les fichiers de LittleFS et les télécharger ;
- téléverser un ou plusieurs fichiers, avec glisser-déposer dans l’interface enrichie ;
- prévisualiser éventuellement quelques formats courants ;
- accéder au service sans connexion Internet ;
- retrouver un mode de secours minimal et robuste.

### Choix retenus

- **Serveur HTTP + petite WebUI spécialisée** : une solution adaptée au besoin.
- **Pas d’USB Mass Storage** : MSC expose un périphérique bloc et ne convient pas directement à LittleFS.
- **Pas de WebDAV** : jugé inutilement complexe pour cette application.

## 🗂️ Organisation de LittleFS

LittleFS est considéré comme un **espace de noms plat**. Exemples de fichiers :

```text
/config.json
/calibration.bin
/log.txt
/.sys_app.js.gz
/.sys_app.css.gz
/.u_<UUID>
```

| Préfixe  | Rôle                                  | Interface enrichie   | Mode recovery |
|----------|---------------------------------------|---------------------:|--------------:|
| `.sys_*` | Ressources système de la WebUI        | Masquées normalement | Affichées     |
| `.u_*`   | Fichiers temporaires de téléversement | Masqués normalement  | Affichés      |
| Autres   | Fichiers utilisateur                  | Affichés             | Affichés      |

## 🖥️ Interface Web

### WebUI enrichie dans LittleFS

Le code client est stocké dans LittleFS afin de pouvoir être mis à jour indépendamment du firmware. Par exemple :

```text
/.sys_app.js.gz
/.sys_app.css.gz
```

Les ressources sont compressées en gzip avant leur déploiement. L’ESP32 **ne les décompresse pas** : il transmet les octets compressés avec les en-têtes appropriés, et le navigateur les décompresse nativement.

```http
Content-Type: application/javascript
Content-Encoding: gzip
Content-Length: <taille compressée>
```

Pour le CSS, utiliser `Content-Type: text/css`. Aucune CDN ni ressource Internet ne doit être nécessaire. Privilégier HTML, CSS et JavaScript natifs, sans jQuery ni framework.

### Page minimale embarquée

Le firmware embarque un petit `index.htm`, idéalement en `PROGMEM`. Il sert à la fois de bootstrap, d’interface de secours et d’interface fonctionnelle lorsque JavaScript est désactivé : c’est le principe de **progressive enhancement**.

Même sans JavaScript, cette page doit permettre de :

- lister tous les fichiers, y compris `.sys_*` et `.u_*` ;
- télécharger chaque fichier avec un lien HTML ;
- téléverser un ou plusieurs fichiers via un formulaire HTML standard `multipart/form-data`.

La liste est générée directement côté ESP32 dans le HTML.

### Passage à l’interface enrichie

Un minuscule bootstrap JavaScript dans `index.htm` tente de charger les ressources suivantes :

```text
/.sys_app.js.gz
/.sys_app.css.gz
```

L’interface riche ne prend le relais qu’après son initialisation réussie. **Ne jamais masquer ni supprimer l’interface de secours avant cette confirmation.**

| Situation | Comportement attendu |
|---|---|
| JavaScript désactivé | Interface minimale |
| LittleFS vide | Interface minimale |
| `/.sys_app.js.gz` absent ou invalide | Interface minimale |
| Application riche initialisée correctement | Interface enrichie |

Une option `/?recovery=1` peut forcer le mode minimal et empêcher le démarrage de l’interface enrichie.

## 🔌 Backend HTTP et API

L’interface enrichie et le fallback doivent utiliser autant que possible le **même backend**. Les fonctions prévues sont :

- liste et téléchargement de fichiers ;
- téléversement et, éventuellement, suppression depuis l’interface riche ;
- informations sur LittleFS et le système ;
- synchronisation temporelle.

Le fallback peut utiliser `multipart/form-data`. L’interface enrichie peut utiliser des téléversements `PUT` en streaming, plus simples et performants. Les deux chemins convergent vers le même moteur transactionnel.

## 📤 Téléversements fiables

Ne jamais écrire directement dans le fichier de destination. Chaque téléversement suit une transaction :

1. Générer un UUID ou un identifiant aléatoire de 64 bits.
2. Créer un fichier temporaire, par exemple `/.u_91A72F730D8B3A11`.
3. Recevoir le flux HTTP et écrire progressivement dans ce fichier.
4. Vérifier la fin du transfert et, si elle est connue, la taille attendue.
5. Appeler `flush()`, puis fermer le fichier.
6. Renommer le temporaire vers sa destination et mettre à jour le catalogue RAM.

En cas de coupure réseau ou de redémarrage, la version précédente du fichier de destination doit rester intacte. Les fichiers temporaires résiduels peuvent être repérés et supprimés au démarrage. Il faut également définir une stratégie pour les téléversements simultanés vers le même nom.

## ⚡ Performances et concurrence

Une piste consiste à utiliser deux buffers PSRAM par téléversement : **2 × 32 Kio** ou **2 × 64 Kio**. L’architecture peut s’appuyer sur :

- plusieurs producteurs HTTP et sessions de téléversement ;
- le double buffering ;
- des files FreeRTOS bornées ;
- une contre-pression (*back-pressure*) lorsque les buffers sont occupés ;
- un seul writer LittleFS sérialisé.

Les deux cœurs de l’ESP32-S3 peuvent découpler le réseau du stockage, mais les écritures Flash ne doivent pas être parallélisées : **Flash et LittleFS restent le goulot d’étranglement partagé.**

## 🔐 SHA-256

Le SHA-256 n’est pas requis pour l’intégrité normale de HTTP/TCP : TCP gère le transport, l’ordre, les retransmissions et la détection des erreurs de transmission. Il peut néanmoins être calculé au fil de l’eau, puisque les données passent déjà par les buffers, et servir de métadonnée facultative pour :

- l’affichage et le diagnostic ;
- la comparaison ou une éventuelle déduplication ;
- un futur contrôle de bout en bout, si le navigateur calcule aussi le SHA attendu.

Le SHA d’un fichier téléversé pendant l’uptime peut être connu immédiatement. Pour les fichiers découverts au démarrage, utiliser `hashValid = false` et ne calculer le hash qu’à la demande.

## 📚 Catalogue des fichiers

Au démarrage, scanner LittleFS et construire un catalogue en RAM. Il n’est pas nécessaire de créer un index persistant supplémentaire pour la première version : **LittleFS reste la source de vérité**, et le catalogue est une vue enrichie volatile.

```cpp
FileInfo {
  name,
  size,
  modifiedUtcMs,
  sha256,
  hashValid
}
```

## 🕒 Gestion du temps

L’ESP32 ne travaille qu’en **UTC**. Le navigateur fournit l’heure Unix en millisecondes avec `Date.now()` ; par exemple via `POST /api/time` et un champ `utcMs`. Le firmware peut ensuite appeler `settimeofday()`.

| Couche | Règle |
|---|---|
| Stockage et protocole | UTC |
| Contrôleur ESP32 | UTC uniquement, sans fuseau ni heure d’été |
| Présentation locale | Conversion dans le navigateur |

Pour afficher une date localement, le navigateur peut utiliser `new Date(timestamp).toLocaleString()`.

### Horodatages volatils

Au démarrage, les fichiers découverts ont `modifiedUtcMs = 0`, ce qui signifie **date inconnue** : ne pas afficher cette valeur comme le 1er janvier 1970. Lorsqu’un fichier est modifié après synchronisation de l’heure, le catalogue RAM reçoit `modifiedUtcMs = nowUtcMs()`. Cette date reste volatile et redevient inconnue après redémarrage. Si le fichier est modifié avant la synchronisation, sa date reste `0`.

## 🧑‍💻 Répartition des responsabilités

| Navigateur | ESP32 |
|---|---|
| Expérience utilisateur et rendu | Wi-Fi / SoftAP et serveur HTTP |
| Glisser-déposer et téléversements multiples | LittleFS et API |
| Progression, tri et filtrage | Téléversements, téléchargements et transactions |
| Formatage des tailles et dates locales | Catalogue RAM et synchronisation UTC |
| Prévisualisation et éventuellement SHA côté client | Recovery et buffers PSRAM |

## 👀 Prévisualisation

Rester volontairement léger pour la première version :

| Format | Prévisualisation envisagée |
|---|---|
| PNG, JPEG, GIF, WebP | Image native du navigateur |
| TXT, LOG, CSV | Texte dans un élément `<pre>` |
| JSON | `JSON.parse()` puis `JSON.stringify(..., null, 2)` |
| XML | Texte brut ou `DOMParser` natif |
| TOML, YAML | Texte brut |
| PDF, BIN, ZIP, etc. | Téléchargement uniquement |

Ne pas embarquer de bibliothèque lourde uniquement pour obtenir des prévisualisations sophistiquées.

## 🏗️ Architecture cible

| Composant | Responsabilités et contenu |
|---|---|
| **Firmware / flash programme** | Serveur HTTP, API, LittleFS, moteur upload/download, transactions, catalogue RAM, gestion UTC et `index.htm` minimal/recovery en `PROGMEM` |
| **LittleFS** | Fichiers utilisateur, `/.sys_app.js.gz`, `/.sys_app.css.gz`, autres ressources WebUI compressées et temporaires `.u_*` |
| **PSRAM** | Buffers de transfert, notamment le double buffering des téléversements |
| **Navigateur** | Application enrichie, rendu, prévisualisation, glisser-déposer, concurrence des téléversements et présentation locale des dates |

> **Principe directeur :** l’interface minimale est déjà pleinement fonctionnelle en HTML pur. JavaScript ne la remplace pas : il l’enrichit lorsqu’il est disponible.

## ➡️ Prochaine étape

Transformer cette architecture en spécification technique, puis en prototype fonctionnel **C++ pour ESP32-S3** avec HTML/CSS/JavaScript, en vérifiant précisément les contraintes des API Arduino-ESP32 et/ou ESP-IDF utilisées.
