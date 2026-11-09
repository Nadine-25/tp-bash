# TP N°2 : Linux & Scripting Bash

> Module **DevOps**, Département d'informatique, Faculté des Sciences Exactes
> Université Abderrahmane Mira, Béjaïa
> Auteur : Nadine (GitHub : [Nadine-25](https://github.com/Nadine-25))

## Sommaire

1. [Objectifs pédagogiques](#objectifs-pédagogiques)
2. [Structure du dépôt](#structure-du-dépôt)
3. [Prérequis](#prérequis)
4. [Exercice 2.0 : Dépôt GitHub](#exercice-20--mise-en-place-du-dépôt-github)
5. [Exercice 2.1 : Navigation, recherche et permissions](#exercice-21--navigation-recherche-et-permissions)
6. [Exercice 2.2 : check_disk.sh](#exercice-22--check_disksh)
7. [Exercice 2.3 : backup.sh](#exercice-23--backupsh)
8. [Exercice 2.4 : Planification avec cron](#exercice-24--planification-avec-cron)
9. [Exercice 2.5 : ci.sh](#exercice-25--cish)
10. [Codes de sortie utilisés](#codes-de-sortie-utilisés)
11. [Vérification du livrable](#vérification-du-livrable)
12. [Réponses aux questions de réflexion](#réponses-aux-questions-de-réflexion)

## Objectifs pédagogiques

- Se repérer dans un système de fichiers Linux et manipuler les permissions
- Écrire des scripts Bash (variables, conditions, boucles, fonctions)
- Automatiser une tâche récurrente et la planifier avec cron
- Comprendre les codes de retour et la gestion d'erreurs
- Versionner son travail avec Git et le publier sur GitHub

## Structure du dépôt

```
tp-bash/
├── .gitignore            # exclut archives, fichiers temporaires d'éditeurs
├── README.md             # ce document
├── REPONSES.md           # réponses aux questions de réflexion
├── check_disk.sh         # Ex 2.2 : alerte d'espace disque
├── backup.sh             # Ex 2.3 : sauvegarde horodatée + purge
├── ci.sh                 # Ex 2.5 : mini-pipeline CI
├── backup.log            # journal d'une exécution réelle de backup.sh
├── ci.log                # journal d'une exécution de ci.sh
├── crontab_export.txt    # export de `crontab -l` (preuve de la tâche cron)
├── logs/                 # Ex 2.1 : fichiers .log de test
└── data/                 # Ex 2.1 : données de test (source des sauvegardes)
```

## Prérequis

- Linux (testé sous Ubuntu/Debian) avec Bash 4+
- Outils : `git`, `tar`, `find`, `grep`, `awk`, `df`, `cron`
- Un compte GitHub et l'accès au dépôt distant `tp-bash`

## Exercice 2.0 : Mise en place du dépôt GitHub

**Objectif :** créer un dépôt distant vide, le lier à un dépôt local et versionner chaque exercice.

```bash
mkdir tp-bash && cd tp-bash
git init -b main
git remote add origin https://github.com/Nadine-25/tp-bash.git
# création du .gitignore, puis :
git add .gitignore
git commit -m "Ex 2.0 : initialisation du dépôt et .gitignore"
git push -u origin main
```

Le `.gitignore` exclut `backups/`, `*.tar.gz`, `*.swp` et `*~`. Les `.log` ne sont volontairement **pas** ignorés, car `backup.log` fait partie du livrable. Un commit au minimum est fait par exercice, avec un message explicite.

## Exercice 2.1 : Navigation, recherche et permissions

| Étape | Commande | Rôle |
|---|---|---|
| Arborescence | `mkdir -p logs data` | Crée les dossiers (sans erreur s'ils existent) |
| Lister les logs | `ls -lh logs/*.log` | Format long, tailles lisibles |
| Fichiers anciens | `find . -type f -mtime +7` | Fichiers modifiés il y a plus de 7 jours |
| Recherche | `grep -ri "erreur" --include="*.log" .` | Récursif, insensible à la casse, limité aux `.log` |
| Ajout du mot | `echo "erreur" >> logs/app1.log` | `>>` ajoute sans écraser |
| Permissions | `chmod 700 script.sh` | Exécutable uniquement par le propriétaire |

Notes :
- `touch -d "10 days ago" fichier` permet de simuler des fichiers anciens pour tester `find -mtime`.
- `grep` renvoie le code 1 quand il ne trouve rien : ce n'est pas une panne, mais cela compte sous `set -e`.
- `chown utilisateur:groupe fichier` change le propriétaire (réservé à root pour donner un fichier à autrui).

## Exercice 2.2 : check_disk.sh

**Rôle :** afficher une alerte si l'espace disque de `/` dépasse un seuil (80 % par défaut).

```bash
chmod +x check_disk.sh
./check_disk.sh        # seuil par défaut : 80 %
./check_disk.sh 10     # seuil bas : provoque une alerte
```

**Fonctionnement :**
1. `SEUIL="${1:-80}"` : premier argument, sinon 80.
2. Validation : le seuil doit être un entier (sinon code 2).
3. `df -P / | awk 'NR==2 {print $5}' | tr -d '%'` : extrait le pourcentage d'utilisation.
4. Comparaison avec `[ "$UTILISATION" -ge "$SEUIL" ]`.

**Codes de sortie :** `0` = OK, `1` = seuil dépassé, `2` = argument invalide.

## Exercice 2.3 : backup.sh

**Rôle :** archiver un dossier dans une archive `.tar.gz` horodatée, puis supprimer les sauvegardes de plus de 7 jours.

```bash
./backup.sh <dossier_a_sauvegarder> [dossier_destination]
# exemple
./backup.sh ./data ~/backups >> backup.log 2>&1
```

**Points clés :**
- `set -Eeuo pipefail` : le script s'arrête à la première erreur, sur variable non définie ou sur échec dans un pipe.
- Un `trap ... ERR` affiche la ligne de l'échec.
- Les erreurs sont envoyées sur stderr (`>&2`).
- Nom d'archive : `<dossier>_AAAA-MM-JJ_HH-MM-SS.tar.gz`.
- Boucle `for` sur les archives et suppression de celles de plus de 7 jours (`find -mtime +7`).

**Codes de sortie :** `0` = succès, `1` = dossier inexistant, `2` = mauvais usage, autre = erreur système interceptée par `set -e`.

## Exercice 2.4 : Planification avec cron

Tâche installée avec `crontab -e` (**chemins absolus obligatoires**) :

```
0 2 * * * /chemin/absolu/tp-bash/backup.sh /chemin/absolu/tp-bash/data /chemin/absolu/backups >> /chemin/absolu/tp-bash/backup.log 2>&1
```

| Champ | Valeur | Signification |
|---|---|---|
| minute | `0` | à la minute 0 |
| heure | `2` | à 2 h |
| jour du mois | `*` | tous les jours |
| mois | `*` | tous les mois |
| jour de semaine | `*` | tous les jours de la semaine |

**Vérification :**

```bash
crontab -l                    # affiche la tâche planifiée
crontab -l > crontab_export.txt
tail -f backup.log            # suivi en direct
```

Pour tester sans attendre 2 h du matin, remplacer temporairement par `* * * * *`, puis remettre `0 2 * * *`.

## Exercice 2.5 : ci.sh

**Rôle :** simuler un pipeline CI minimal.

```bash
./ci.sh https://github.com/Nadine-25/tp-bash.git
```

**Étapes :**
1. Récupération du code : `git clone` (première fois) ou `git pull --ff-only`.
2. Tests simulés : présence du `.gitignore` et validation syntaxique de chaque `.sh` (`bash -n`).
3. Journalisation horodatée dans `ci.log` via `tee -a`, avec une fonction `log()`.
4. Code de sortie explicite.

**Codes de sortie :** `0` = pipeline réussi, `1` = tests en échec, `2` = usage, `3` = échec clone/pull.

## Codes de sortie utilisés

| Code | Signification | Script |
|---|---|---|
| 0 | Succès | tous |
| 1 | Seuil dépassé / dossier absent / tests en échec | check_disk, backup, ci |
| 2 | Mauvais usage ou argument invalide | tous |
| 3 | Échec de récupération du code | ci |

Le code de sortie d'une commande se lit avec `echo $?`.

## Vérification du livrable

```bash
ls -l                 # check_disk.sh, backup.sh, ci.sh avec le droit x
crontab -l            # tâche cron visible
cat backup.log        # exécution réelle de backup.sh
cat REPONSES.md       # réponses aux questions de réflexion
git log --oneline     # au moins un commit par exercice
```

## Réponses aux questions de réflexion

Les réponses détaillées sont dans [REPONSES.md](REPONSES.md). Résumé :

| Exercice | Question | Réponse courte |
|---|---|---|
| 2.1 | `chmod 644` ? | `rw-r--r--` ; non exécutable, même par le propriétaire (aucun bit `x`) |
| 2.2 | Pourquoi `"$UTILISATION"` ? | Évite le word splitting et le globbing ; le test garde un nombre d'arguments correct même si la variable est vide |
| 2.3 | Sans `set -Eeuo pipefail` ? | Le script continue après l'échec de `tar`, affiche un faux succès, purge les anciennes sauvegardes et renvoie 0 |
| 2.4 | Pourquoi des chemins absolus ? | Cron utilise un environnement minimal (répertoire courant = `$HOME`, `PATH` réduit, pas de `.bashrc`) |
| 2.5 | Vers un vrai CI ? | Déclencheur automatique, environnement isolé, dépendances, lint, tests, build, déploiement, notifications ; un code de retour distinct par point de défaillance |

## Références

- `man bash`, `man find`, `man grep`, `man crontab`, `man chmod`
- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
- [crontab.guru](https://crontab.guru/) pour vérifier une expression cron
