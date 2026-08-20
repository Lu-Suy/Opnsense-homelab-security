
**Projet :** Bastion Godmode / Horus AIS  
**Date :** 20 août 2026  
**Machine :** G4 (Rocky Linux 10)  
**Objectif :** Mettre en place une rotation et une rétention automatiques des backups


**Standards / bonnes pratiques :**

- On ne parle pas vraiment d’« ANSI » pour les rétentions de backup.
- Les références professionnelles sont plutôt :
    - **Règle 3-2-1** (3 copies, 2 supports différents, 1 offsite)
    - NIST SP 800-34 / ISO 27001 (continuité d’activité)
    - Grandfather-Father-Son (très courant en entreprise)


On fige donc :

- **Daily** → on garde les **7 derniers jours**
- **Weekly** → on garde les **4 dernières semaines**
- OPNsense → déjà géré par le plugin (30 configs)

---

### Structure de dossiers que je propose

```text
/mnt/backup_g4/
├── Daily/                  ← les 7 derniers backups complets
│   ├── G4_Rocky_2026-08-14_03-00-01/
│   ├── G4_Rocky_2026-08-15_03-00-01/
│   └── ...
├── Weekly/                 ← les 4 derniers "weekly" (ex: dimanche)
│   ├── G4_Rocky_2026-08-03_03-00-01/
│   └── ...
├── OPNsense/               ← inchangé
├── Scripts/                ← scripts + logs
│   ├── backup_g4.sh
│   ├── rotate_backups.sh   ← nouveau
│   └── backup_g4.log
└── Archives/               ← optionnel (pour plus tard)
```

### Principe de la rotation (simple et robuste)

1. Le script de backup (`backup_g4.sh`) continue de créer un nouveau dossier **dans Daily/** tous les jours à 03h00.
2. Juste après (ou dans un second cron), le script `rotate_backups.sh` :
   - Garde uniquement les **7 plus récents** dans `Daily/`
   - Si un backup tombe un **dimanche**, il en fait une copie (ou un hardlink) dans `Weekly/`
   - Dans `Weekly/`, il ne garde que les **4 plus récents**
   - Supprime tout le reste proprement
1. On réduit aussi la verbosité de rsync pour que le log ne grossisse plus aussi vite.

---

### Étape 1 – Création de la nouvelle structure de dossiers

On va juste créer les deux nouveaux dossiers Daily et Weekly à la racine du volume de backup.

Bash

```
# Création des dossiers
sudo mkdir -p /mnt/backup_g4/Daily
sudo mkdir -p /mnt/backup_g4/Weekly

# Vérification
ls -la /mnt/backup_g4/
```

**Ce que ça fait :**

- mkdir -p crée les dossiers s’ils n’existent pas (et ne plante pas s’ils existent déjà).
- On reste en root pour l’instant (les permissions seront gérées plus tard si besoin).

![[Pasted image 20260820150903.png]]


Etape 1 validée.

Les dossiers `Daily/` et `Weekly/` sont bien créés.

---

### Étape 2 – On range d’abord les backups existants

Avant de modifier le script, on va déplacer les 4 backups actuels de `G4_Rocky/` vers `Daily/` pour que tout soit propre dès le départ.

Exécute ces commandes :

```bash
# Déplacement des backups existants dans Daily/
sudo mv /mnt/backup_g4/G4_Rocky/G4_Rocky_* /mnt/backup_g4/Daily/
# Déplacement forcé via bash (évite le problème zsh) 
sudo bash -c 'mv /mnt/backup_g4/G4_Rocky/G4_Rocky_* /mnt/backup_g4/Daily/'

# Vérification
ls -la /mnt/backup_g4/Daily/
ls -la /mnt/backup_g4/G4_Rocky/
```

**Ce que ça fait :**
- On déplace les 4 dossiers horodatés dans le nouveau dossier `Daily/`.
- Le dossier `G4_Rocky/` devrait se retrouver vide (on pourra le supprimer après si tu veux).

![[Pasted image 20260820151533.png]]
![[Pasted image 20260820151602.png]]


Déplacement réussi.

Les 4 backups sont maintenant dans `Daily/` et le dossier `G4_Rocky/` est vide.

---

### Étape 3 – Modification du script `backup_g4.sh`

On va faire deux changements importants :

1. Les nouveaux backups iront désormais dans **`Daily/`**
2. On réduit fortement la verbosité de rsync (le log ne grossira plus de 100+ Mo à chaque run)

 Contenu complet du script (propre et commenté) :

```bash
#!/bin/bash

# ============================================================
# Script de backup de la G4 vers le volume chiffré
# Projet : Bastion Godmode / Horus AIS
# Date   : 17 août 2026 (mis à jour 20 août 2026 – 36B)
# ============================================================

BACKUP_ROOT="/mnt/backup_g4"
BACKUP_DIR="${BACKUP_ROOT}/Daily"
LOG_FILE="${BACKUP_ROOT}/Scripts/backup_g4.log"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
TARGET="${BACKUP_DIR}/G4_Rocky_${DATE}"

# Vérification que le volume est monté
if ! mountpoint -q "$BACKUP_ROOT"; then
    echo "[$(date)] ERREUR : Le volume $BACKUP_ROOT n'est pas monté." | tee -a "$LOG_FILE"
    exit 1
fi

mkdir -p "$TARGET"

echo "[$(date)] Début du backup de la G4 vers $TARGET" | tee -a "$LOG_FILE"

# -aAX = archive + ACLs + xattrs
# --info=stats2 = statistiques utiles sans lister tous les fichiers
# --delete = miroir exact
rsync -aAX --info=stats2 --delete \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/home/*/.cache"} \
  / "$TARGET/" >> "$LOG_FILE" 2>&1

RSYNC_EXIT=$?

if [ $RSYNC_EXIT -eq 0 ] || [ $RSYNC_EXIT -eq 23 ]; then
    echo "[$(date)] Backup terminé avec succès (code $RSYNC_EXIT)" | tee -a "$LOG_FILE"
    echo "[$(date)] Taille : $(du -sh "$TARGET" | cut -f1)" | tee -a "$LOG_FILE"
    exit 0
else
    echo "[$(date)] ERREUR : le backup a échoué (code $RSYNC_EXIT)" | tee -a "$LOG_FILE"
    exit $RSYNC_EXIT
fi
```

**Pour l’appliquer :**

```bash
sudo nano /mnt/backup_g4/Scripts/backup_g4.sh
```

- Effacer tout le contenu actuel
- Coller le nouveau script ci-dessus
- Sauvegarder (`Ctrl+O` puis `Entrée`, puis `Ctrl+X`)

Puis vérifier :

```bash
# Vérifier que le script est bien à jour
head -n 25 /mnt/backup_g4/Scripts/backup_g4.sh

# S’assurer qu’il est toujours exécutable
sudo chmod +x /mnt/backup_g4/Scripts/backup_g4.sh
```


---

### Étape 4 – Création du script de rotation `rotate_backups.sh`

Créer un script propre qui :

1. Vérifier que le volume est monté
2. Dans `Daily/` → garder uniquement les **7 plus récents**
3. Si un backup tombe un **dimanche**, le copie dans `Weekly/`
4. Dans `Weekly/` → garde uniquement les **4 plus récents**
5. Loggue tout proprement

Voici le script complet :

```bash
sudo tee /mnt/backup_g4/Scripts/rotate_backups.sh > /dev/null << 'EOF'
#!/bin/bash

# ============================================================
# Script de rotation des backups G4
# Projet : Bastion Godmode / Horus AIS
# Date   : 20 août 2026 – Fiche 36B
# ============================================================

BACKUP_ROOT="/mnt/backup_g4"
DAILY_DIR="${BACKUP_ROOT}/Daily"
WEEKLY_DIR="${BACKUP_ROOT}/Weekly"
LOG_FILE="${BACKUP_ROOT}/Scripts/rotate_backups.log"
KEEP_DAILY=7
KEEP_WEEKLY=4

# Vérification volume monté
if ! mountpoint -q "$BACKUP_ROOT"; then
    echo "[$(date)] ERREUR : Le volume $BACKUP_ROOT n'est pas monté." | tee -a "$LOG_FILE"
    exit 1
fi

echo "[$(date)] === Début de la rotation des backups ===" | tee -a "$LOG_FILE"

# --- 1. Rotation Daily (garder les 7 plus récents) ---
cd "$DAILY_DIR" || exit 1

# Liste des backups triés du plus récent au plus ancien
mapfile -t DAILY_BACKUPS < <(ls -1d G4_Rocky_* 2>/dev/null | sort -r)

COUNT_DAILY=${#DAILY_BACKUPS[@]}
echo "[$(date)] Nombre de backups Daily trouvés : $COUNT_DAILY" | tee -a "$LOG_FILE"

if [ "$COUNT_DAILY" -gt "$KEEP_DAILY" ]; then
    TO_DELETE=$((COUNT_DAILY - KEEP_DAILY))
    echo "[$(date)] Suppression de $TO_DELETE ancien(s) backup(s) Daily..." | tee -a "$LOG_FILE"
    for ((i=KEEP_DAILY; i<COUNT_DAILY; i++)); do
        echo "[$(date)] Suppression : ${DAILY_BACKUPS[$i]}" | tee -a "$LOG_FILE"
        rm -rf "${DAILY_BACKUPS[$i]}"
    done
else
    echo "[$(date)] Aucun backup Daily à supprimer (≤ $KEEP_DAILY)" | tee -a "$LOG_FILE"
fi

# --- 2. Promotion vers Weekly (si dimanche) ---
# On regarde le backup le plus récent
LATEST="${DAILY_BACKUPS[0]}"
if [ -n "$LATEST" ]; then
    # Extraire la date du nom (format G4_Rocky_YYYY-MM-DD_...)
    BACKUP_DATE=$(echo "$LATEST" | grep -oP '\d{4}-\d{2}-\d{2}')
    if [ -n "$BACKUP_DATE" ]; then
        # Vérifier si c'était un dimanche (1 = lundi ... 7 = dimanche avec date)
        DAY_OF_WEEK=$(date -d "$BACKUP_DATE" +%u)
        if [ "$DAY_OF_WEEK" -eq 7 ]; then
            echo "[$(date)] Backup du dimanche détecté : $LATEST → copie vers Weekly/" | tee -a "$LOG_FILE"
            cp -a "$LATEST" "$WEEKLY_DIR/"
        fi
    fi
fi

# --- 3. Rotation Weekly (garder les 4 plus récents) ---
cd "$WEEKLY_DIR" || exit 1

mapfile -t WEEKLY_BACKUPS < <(ls -1d G4_Rocky_* 2>/dev/null | sort -r)

COUNT_WEEKLY=${#WEEKLY_BACKUPS[@]}
echo "[$(date)] Nombre de backups Weekly trouvés : $COUNT_WEEKLY" | tee -a "$LOG_FILE"

if [ "$COUNT_WEEKLY" -gt "$KEEP_WEEKLY" ]; then
    TO_DELETE=$((COUNT_WEEKLY - KEEP_WEEKLY))
    echo "[$(date)] Suppression de $TO_DELETE ancien(s) backup(s) Weekly..." | tee -a "$LOG_FILE"
    for ((i=KEEP_WEEKLY; i<COUNT_WEEKLY; i++)); do
        echo "[$(date)] Suppression : ${WEEKLY_BACKUPS[$i]}" | tee -a "$LOG_FILE"
        rm -rf "${WEEKLY_BACKUPS[$i]}"
    done
else
    echo "[$(date)] Aucun backup Weekly à supprimer (≤ $KEEP_WEEKLY)" | tee -a "$LOG_FILE"
fi

echo "[$(date)] === Fin de la rotation des backups ===" | tee -a "$LOG_FILE"
EOF
```

Puis le rendre exécutable et vérifier :

```bash
sudo chmod +x /mnt/backup_g4/Scripts/rotate_backups.sh
ls -l /mnt/backup_g4/Scripts/rotate_backups.sh
head -n 20 /mnt/backup_g4/Scripts/rotate_backups.sh
```

![[Pasted image 20260820152552.png]]

Le script de rotation est bien en place.

---

### Étape 5 – Premier test manuel de la rotation

Comme on n’a que **4 backups** dans `Daily/` (on en garde 7), le script ne doit **rien supprimer**. C’est exactement ce qu’on veut pour un premier test.

Le lancer manuellement :

```bash
sudo /mnt/backup_g4/Scripts/rotate_backups.sh
```

Puis regarder le log :

```bash
sudo cat /mnt/backup_g4/Scripts/rotate_backups.log
```

Et vérifier l’état des dossiers :

```bash
ls -la /mnt/backup_g4/Daily/
ls -la /mnt/backup_g4/Weekly/
```

Le script de rotation fonctionne exactement comme prévu :
- 4 backups Daily trouvés → rien de supprimé
- Weekly vide → rien de supprimé
- Log propre

---

### Étape 6 – Ajout du cron pour la rotation

On va lancer la rotation **juste après** le backup (à 03h15).

Exécuter :

```bash
sudo crontab -e
```

Ajouter cette ligne (en dessous de la ligne du backup) :

```bash
15 3 * * * /mnt/backup_g4/Scripts/rotate_backups.sh
```

Sauvegarder et quitter.

Puis vérifier :

```bash
sudo crontab -l
```

![[Pasted image 20260820153320.png]]

Le crontab est maintenant correct :

### Récapitulatif technique – Fiche 36B (terminé)

| Élément                    | Statut | Détail                                   |
| -------------------------- | ------ | ---------------------------------------- |
| Structure Daily / Weekly   | ✅      | Créée                                    |
| Anciens backups            | ✅      | Déplacés dans `Daily/`                   |
| Script `backup_g4.sh`      | ✅      | Destination = `Daily/` + `--info=stats2` |
| Script `rotate_backups.sh` | ✅      | Créé + testé (comportement correct)      |
| Cron                       | ✅      | Backup 03:00 + Rotation 03:15            |
| Politique de rétention     | ✅      | **7 Daily + 4 Weekly**                   |

**Petit nettoyage optionnel :**

```bash
sudo rmdir /mnt/backup_g4/G4_Rocky
```

(le dossier est vide)

---
### 6. Clôture

La politique de rotation et de rétention des backups G4 est désormais opérationnelle.

- Structure Daily/ + Weekly/ en place
- Script de backup mis à jour (destination + réduction de verbosité)
- Script de rotation créé, testé et planifié
- Cron configuré (backup 03h00 + rotation 03h15)
- Politique validée : **7 daily + 4 weekly**

Les prochains backups quotidiens seront automatiquement classés et purgés selon la politique définie.

**Points de vigilance restants (non bloquants) :**

- Surveiller les premières rotations automatiques via le log rotate_backups.log
- Version rsync 3.4.4 (attendre 3.5.0 dans les repos Rocky)
- Contexts SELinux unlabeled_t (possibilité de restorecon ultérieur)

**Fin de la fiche 36B – 20 août 2026**