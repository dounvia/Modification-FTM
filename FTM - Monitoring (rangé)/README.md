# FTM - Monitoring

Page web qui supervise les nœuds Secure Transport des environnements hors production (QUA, DEV, INT).
L'outil contrôle chaque nœud toutes les 60 secondes, affiche l'état sur une page web et envoie une alerte par mail quand un service s'arrête.

Python 3.6 ou plus récent, aucune bibliothèque à installer.

## Lancer l'outil

```
python3 supervision.py            # vraies machines
python3 supervision.py --demo     # machines fictives, pour tester
```

La page est ensuite disponible sur `http://<serveur>:8080`.

Autres options :

```
python3 supervision.py --test-mail      # envoie une alerte d'exemple
python3 supervision.py --test-rapport   # envoie le rapport quotidien
python3 supervision.py --port 8081      # change le port de la page
```

Pour l'arrêter : `Ctrl + C`, ou `sudo systemctl stop ftm-monitoring` s'il tourne en service.

## Dossiers

| Dossier | Contenu |
|---|---|
| `supervision.py` | le fichier à lancer |
| `ftm/` | le code Python |
| `config/` | les réglages (fichiers CSV) |
| `donnees/` | `etat.json`, écrit par l'outil (historique, journal, maintenance) |
| `web/` | la page web : `index.html`, `style.css`, `app.js`, `bip.js`, polices, images des mails |
| `deploiement/` | le service systemd |
| `tests/` | les tests automatiques |

Code Python (`ftm/`) :

| Fichier | Rôle |
|---|---|
| `config.py` | chemins, réglages, lecture des CSV |
| `verification.py` | contrôle des nœuds par SSH (`ps`) et test des ports |
| `superviseur.py` | boucle de contrôle, `etat.json`, alertes, maintenance, journal, rapport |
| `mails.py` | mails d'alerte et rapport quotidien (SMTP ou Outlook) |
| `serveur_web.py` | page web et API |

## Réglages (`config/`)

### environnements.csv : les nœuds à surveiller

Une ligne par nœud. Copier `environnements.exemple.csv` pour partir d'un modèle.

```
env;type;noeud;hostname;ip;user_ssh;url;cluster
```

Ajouter un nœud = ajouter une ligne. Pas besoin de toucher au code.

### services.csv : ce qu'on contrôle sur chaque nœud

- `acces` : test d'un port (SSH 22, IHM 8444) ;
- `service` : commande lancée par SSH. Le service est démarré si la commande réussit.

Les services sont repérés avec `ps -eo args | grep -qE -- '-[D]ct=<nom>( |$)'`.
On utilise `ps` et pas `pgrep`, car `pgrep` ne lit pas la fin des lignes de lancement très longues (cas du Transaction Manager).

### alertes.csv : les mails

| Paramètre | Valeur |
|---|---|
| `actif` | `oui` ou `non` |
| `methode` | `smtp` (serveur Linux) ou `outlook` (poste Windows) |
| `serveur_smtp`, `port` | serveur mail SMTP |
| `expediteur`, `destinataires` | adresses (plusieurs destinataires séparés par des virgules) |
| `confirmation` | nombre de contrôles identiques avant d'envoyer une alerte (2 par défaut) |
| `rapport_quotidien` | heure du rapport, par exemple `09:00` |

Si le serveur SMTP demande un mot de passe, il se met dans la variable d'environnement `FTM_SMTP_MOT_DE_PASSE`, jamais dans un fichier.

## Accès aux nœuds

L'outil se connecte en SSH avec une clé, sans mot de passe. Il ne fait que **lire** : aucune commande ne modifie les serveurs.

## Mettre en service sur le serveur

Voir `deploiement/ftm-monitoring.service` (à installer par un administrateur). L'outil redémarre alors tout seul en cas de coupure.

## Tests

```
python3 -m unittest discover -s tests -v
```

## Si un service apparaît arrêté

1. Cliquer sur le nœud pour voir le détail et l'heure de l'arrêt.
2. Vérifier à la main : `ssh <compte>@<ip> "ps -eo args | grep -o -- '-Dct=[a-z]*' | sort -u"`
3. Si l'arrêt est réel, prévenir le référent de la plateforme.
4. Pendant une intervention prévue, mettre le nœud **en maintenance** depuis la page : aucune alerte n'est envoyée.

## Fichiers à ne jamais publier

`config/environnements.csv` (adresses IP), `config/alertes.csv` (adresses mail) et `donnees/` sont exclus par `.gitignore`.
