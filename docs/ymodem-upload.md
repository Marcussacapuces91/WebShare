# Téléversement série YMODEM

Cette spécification ajoute un chemin d’import de fichiers par YMODEM, en complément des téléversements HTTP. Elle vise l’ESP32-S3 et réutilise le même modèle de transaction LittleFS afin qu’une interruption série ne remplace jamais un fichier valide par un fichier incomplet.

## Objectifs

- Recevoir un ou plusieurs fichiers avec un terminal ou un outil compatible YMODEM.
- Fonctionner sans Wi-Fi ni navigateur, notamment en mode de récupération.
- Écrire les données en flux, avec une mémoire bornée, puis publier le fichier uniquement après validation complète.
- Garder un seul writer LittleFS : les écritures série et HTTP sont sérialisées et ne modifient jamais LittleFS en parallèle.

## Interface série

- Le transfert utilise un UART configuré par le firmware (UART USB natif ou port série matériel, selon la carte et le câblage). Le débit et les paramètres série sont configurables ; ils doivent correspondre à ceux du terminal.
- Le mode YMODEM est lancé explicitement par une commande ou une action de récupération afin d’éviter d’interpréter les journaux de démarrage comme une session de transfert.
- Pendant une session, réserver le port à YMODEM : ne pas mélanger les messages de diagnostic avec les octets du protocole. Les diagnostics peuvent être désactivés ou redirigés vers un autre port.
- Le récepteur annonce sa disponibilité selon YMODEM-CRC et gère le démarrage, les blocs de données, les acquittements, les retransmissions et la fin de lot. Les formats de blocs YMODEM pris en charge doivent être documentés et testés avec les terminaux ciblés.

## Réception et validation

1. Lire l’en-tête du fichier YMODEM et extraire le nom et, si fourni, la taille annoncée.
2. Valider le nom : rejeter les chemins, séparateurs, noms vides, noms réservés (`.sys_*`, `.u_*`) et toute valeur incompatible avec l’espace de noms plat de LittleFS. Ne jamais utiliser un chemin fourni par le terminal sans cette validation.
3. Créer un fichier temporaire unique sous `/.u_<identifiant>` ; ne pas ouvrir directement le fichier final.
4. Écrire les blocs reçus progressivement dans le temporaire en utilisant des buffers bornés. Vérifier le CRC du protocole, les numéros de blocs, les retransmissions, les erreurs d’écriture et la taille annoncée lorsqu’elle est disponible. Ne pas compter les octets de remplissage du dernier bloc au-delà de la taille indiquée.
5. En cas d’erreur, d’annulation YMODEM, de dépassement de capacité ou de perte de session, fermer et supprimer le temporaire. L’ancienne version de la destination reste intacte.
6. Après réception et fin de lot valides, vérifier la taille, appeler `flush()`, fermer le fichier, puis publier le résultat via le mécanisme de remplacement atomique retenu pour LittleFS. Mettre ensuite à jour le catalogue RAM et les métadonnées (heure UTC si connue, SHA-256 si calculé).

Si la taille n’est pas présente ou n’est pas exploitable, appliquer une limite maximale configurée et valider le nombre d’octets effectivement reçus. La politique de collision avec un fichier existant doit être explicite : remplacement après succès complet, ou refus ; dans les deux cas, ne jamais supprimer la version existante avant que le nouveau fichier soit validé. Le comportement précis du renommage/remplacement doit être vérifié avec la version de LittleFS utilisée.

## Concurrence et ressources

- Le transfert YMODEM est un producteur supplémentaire pour la file de travail LittleFS, mais il n’introduit pas de writer Flash parallèle.
- Appliquer une contre-pression en ralentissant les acquittements si les buffers ou la file sont pleins ; ne pas acquitter un bloc avant que son contenu soit accepté dans le chemin d’écriture sécurisé.
- Définir des limites de taille, de durée d’inactivité et de nombre de fichiers par lot. Une annulation ou un délai dépassé nettoie les temporaires.
- Les téléversements HTTP concurrents sont soit mis en attente, soit refusés pendant la session série, selon une politique choisie par le firmware. L’accès concurrent au même nom doit être empêché.

## Intégration au projet

YMODEM partage avec HTTP le moteur de réception transactionnel, le writer LittleFS, la validation des noms, la mise à jour du catalogue et le calcul facultatif du SHA-256. Seule la couche de transport diffère. L’interface Web peut indiquer qu’un transfert série est en cours, mais n’est pas nécessaire à son fonctionnement.

## Tests d’acceptation

- Un fichier vide, un fichier binaire et un fichier plus grand que les buffers sont reçus correctement.
- Un lot de plusieurs fichiers est traité selon la sémantique YMODEM choisie.
- CRC invalide, bloc dupliqué, bloc manquant, retransmission, annulation et déconnexion ne publient aucun fichier partiel.
- Une taille annoncée incohérente, un nom avec chemin ou un nom réservé est rejeté sans toucher aux fichiers existants.
- Un stockage plein ou une erreur d’écriture nettoie le temporaire et conserve l’ancienne destination.
- Le comportement de remplacement et de renommage est validé sur la version réelle de LittleFS et les outils YMODEM ciblés.
