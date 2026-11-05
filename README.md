# WebShare
Juste une idée à ce stade, développée avec Copilot

## Prompt de reprise

Oui, le bloc d’écriture semble ne pas s’être rendu correctement. Je te le remets ci-dessous en texte Markdown normal, directement visible et copiable.

Prompt de reprise

Je souhaite reprendre la conception puis l'implémentation d'un gestionnaire de fichiers Web très léger pour ESP32-S3 + LittleFS, utilisable notamment lorsque l'ESP32 fonctionne en Wi-Fi SoftAP sans accès Internet.

1. Objectif

Permettre depuis un navigateur Windows/Linux/macOS/mobile de :

lister les fichiers présents dans LittleFS ;
télécharger un fichier ;
uploader un ou plusieurs fichiers ;
supporter le drag & drop dans l'interface riche ;
éventuellement prévisualiser certains formats simples ;
fonctionner sans aucune connexion Internet ;
disposer d'un mode recovery minimal et robuste.

Nous avons écarté :

USB Mass Storage, car MSC expose un périphérique bloc et ne convient pas directement à LittleFS ;
WebDAV, jugé inutilement complexe par rapport à notre besoin.

Le choix retenu est donc : serveur HTTP + petite WebUI spécialisée.

2. LittleFS comme namespace plat

Considérer LittleFS comme un espace de noms plat.

Exemples :

/config.json
/calibration.bin
/log.txt
/.sys_app.js.gz
/.sys_app.css.gz
/.u_<UUID>

Conventions :

.sys_* = assets système de la WebUI ;
.u_* = fichiers temporaires d'upload.

L'interface riche masque normalement ces fichiers internes.

Le mode recovery les affiche tous.

3. WebUI riche stockée dans LittleFS

Le code client riche doit être stocké dans LittleFS pour pouvoir être mis à jour indépendamment du firmware.

Par exemple :

/.sys_app.js.gz
/.sys_app.css.gz

Les fichiers sont compressés en gzip avant compilation/déploiement.

L'ESP32 ne les décompresse jamais. Il transmet directement les octets gzip avec :

Content-Type: application/javascript ou text/css
Content-Encoding: gzip
Content-Length: taille compressée

Le navigateur fait nativement la décompression.

Aucune CDN ou ressource Internet ne doit être nécessaire.

Utiliser autant que possible HTML/CSS/JavaScript vanilla, sans JQuery ni framework.

4. index.htm minimal embarqué avec le firmware

Le firmware contient en dur, idéalement en PROGMEM, un petit index.htm.

Ce fichier joue simultanément le rôle de :

bootstrap de l'application riche ;
interface de secours ;
interface fonctionnant lorsque JavaScript est désactivé.

Principe : progressive enhancement.

Sans JavaScript, cette page permet déjà :

de lister tous les fichiers ;
de télécharger chaque fichier via un lien HTML ;
d'uploader un ou plusieurs fichiers avec un formulaire HTML standard multipart/form-data.

La liste des fichiers est générée directement côté ESP32 dans le HTML.

Elle affiche tous les fichiers, sans filtre, y compris .sys_* et .u_*.

5. Passage automatique vers l'interface riche

Si JavaScript fonctionne, un bootstrap JS minuscule contenu dans index.htm tente de charger :

/.sys_app.js.gz
/.sys_app.css.gz

Si le chargement et l'initialisation fonctionnent, le JS transforme/prend possession du DOM et affiche l'interface riche.

Ne jamais supprimer ou masquer l'interface recovery tant que l'application riche n'est pas correctement initialisée.

Donc :

JS désactivé → interface minimale ;
LittleFS vide → interface minimale ;
.sys_app.js.gz absent → interface minimale ;
JS corrompu → interface minimale ;
application riche correctement initialisée → interface riche.

Prévoir éventuellement :

/?recovery=1

pour forcer le mode minimal et empêcher le démarrage de l'UI riche.

6. Backend HTTP unique

La WebUI riche et le fallback doivent utiliser le même moteur backend autant que possible.

Fonctions :

listing ;
download ;
upload ;
delete éventuellement dans l'UI riche ;
informations LittleFS ;
synchronisation temporelle ;
informations système.

Le fallback peut utiliser multipart/form-data.

La WebUI riche peut utiliser des uploads PUT streaming plus simples et performants.

Les deux doivent converger vers le même moteur interne d'upload transactionnel.

7. Uploads transactionnels

Ne jamais écrire directement dans le fichier destination.

Pour chaque upload :

générer un UUID ou identifiant aléatoire 64 bits ;
créer par exemple /.u_91A72F730D8B3A11 ;
recevoir le flux HTTP ;
écrire progressivement le temporaire ;
vérifier que le transfert est complet ;
vérifier la taille si elle est connue ;
flush() puis close() ;
renommer vers le fichier définitif ;
mettre à jour le catalogue RAM.

En cas de coupure réseau ou reset, le fichier destination précédent doit rester intact.

Les temporaires résiduels peuvent être détectés et supprimés au démarrage.

Gérer les collisions lorsque deux uploads ciblent simultanément le même nom final.

8. PSRAM, multicœur et uploads multiples

Utiliser éventuellement deux buffers PSRAM par upload :

2 × 32 Kio ;
ou 2 × 64 Kio.

Architecture :

plusieurs producteurs HTTP ;
plusieurs sessions d'upload ;
double buffering ;
queues FreeRTOS bornées ;
back-pressure lorsque les buffers sont occupés ;
un writer LittleFS sérialisé.

Les deux cœurs du S3 permettent de découpler réseau et stockage, mais il ne faut pas chercher à paralléliser physiquement les écritures Flash.

La Flash/LittleFS reste le goulot commun.

9. SHA-256

Le SHA-256 n'est pas nécessaire pour assurer l'intégrité normale de HTTP/TCP.

TCP assure déjà transport, ordre, retransmission et détection des erreurs.

Le SHA peut néanmoins être calculé au fil de l'eau puisque les données passent déjà dans les buffers.

Il devient une métadonnée facultative pour :

affichage ;
diagnostic ;
comparaison ;
déduplication future ;
contrôle E2E futur si le navigateur calcule également le SHA attendu.

Pour un fichier uploadé pendant l'uptime, son SHA peut donc être connu immédiatement.

Pour les fichiers découverts au boot :

hashValid = false

et le SHA peut être calculé seulement à la demande.

10. Catalogue des fichiers en RAM

Au démarrage :

scanner LittleFS ;
construire un catalogue RAM ;
ne pas créer initialement d'index persistant supplémentaire.

Conceptuellement :

FileInfo { name, size, modifiedUtcMs, sha256, hashValid }

LittleFS reste la vérité persistante.

Le catalogue RAM est une vue enrichie volatile.

11. Temps : UTC uniquement sur l'ESP32

Le contrôleur ne travaille qu'en UTC.

Le navigateur fournit l'heure avec :

Date.now()

qui correspond à un Unix Epoch en millisecondes.

Par exemple :

POST /api/time

avec utcMs.

L'ESP32 utilise alors settimeofday().

Aucune timezone ni DST côté firmware.

Règle :

stockage = UTC ;
protocole = UTC ;
contrôleur = UTC ;
conversion locale = uniquement navigateur.

Le navigateur peut utiliser :

new Date(timestamp).toLocaleString()

pour la présentation locale.

12. Timestamps fichiers volatils

Au boot, tous les fichiers découverts ont :

modifiedUtcMs = 0

0 signifie :

date inconnue

Elle n'est donc pas affichée.

Lorsque l'ESP dispose d'une heure valide et qu'un fichier est modifié/uploadé :

modifiedUtcMs = nowUtcMs()

La valeur est conservée uniquement dans le catalogue RAM.

Après reboot, elle redevient 0.

Si une modification intervient avant synchronisation de l'heure, sa date reste 0.

Ne jamais afficher artificiellement le 01/01/1970.

13. Responsabilités du navigateur

Faire effectuer le maximum de travail à la machine cliente :

UX ;
drag & drop ;
uploads multiples ;
progress bars ;
tri ;
filtrage ;
formatage des tailles ;
conversion UTC → heure locale ;
prévisualisation ;
éventuellement SHA côté client.

L'ESP32 conserve les tâches déterministes :

Wi-Fi / SoftAP ;
serveur HTTP ;
LittleFS ;
API ;
uploads/downloads ;
transactions filesystem ;
catalogue RAM ;
temps UTC ;
synchronisation ;
recovery ;
buffers PSRAM.
14. Preview fichiers

Rester volontairement léger.

V1 :

PNG/JPEG/GIF/WebP → image native ;
TXT/LOG/CSV → <pre> ;
JSON → JSON.parse() + JSON.stringify(..., null, 2) ;
XML → texte ou DOMParser natif ;
TOML/YAML → texte brut ;
PDF/BIN/ZIP/etc. → download uniquement.

Ne pas embarquer des librairies lourdes uniquement pour obtenir des previews sophistiquées.

15. Architecture finale

Firmware / flash programme :

serveur HTTP ;
API ;
LittleFS ;
moteur upload/download ;
gestion transactions ;
catalogue RAM ;
gestion UTC ;
index.htm minimal/recovery en PROGMEM.

LittleFS :

fichiers utilisateur ;
.sys_app.js.gz ;
.sys_app.css.gz ;
éventuels autres assets WebUI compressés ;
fichiers temporaires .u_*.

PSRAM :

buffers de transfert ;
notamment double buffers des uploads.

Navigateur :

application riche ;
rendu ;
preview ;
drag & drop ;
concurrence des uploads ;
présentation locale des dates.

Principe directeur :

L'interface minimale est déjà parfaitement fonctionnelle en HTML pur. JavaScript ne remplace pas cette interface : il l'enrichit lorsqu'il est disponible.

La prochaine étape est de transformer cette architecture en spécification technique puis prototype fonctionnel C++ pour ESP32-S3 + HTML/CSS/JS, en vérifiant précisément les contraintes des API Arduino-ESP32/ESP-IDF, du serveur HTTP retenu et de LittleFS.
