
---

**36 – Automatisation des backups (G4 + OPNsense)**

**Projet :** Bastion Godmode / Horus AIS  
**Date :** 17 août 2026  
**Machine concernée :** G4 (Rocky Linux 10) + OPNsense  
**Destination :** Volume chiffré `/mnt/backup_g4`

### 1. Objectif

Mettre en place une automatisation simple, fiable et documentée des sauvegardes vers le volume interne chiffré, afin de :

- Garantir une copie régulière de la G4
- Sauvegarder les configurations critiques d’OPNsense
- Réduire le risque de perte de données
- Poser les bases d’une politique de sauvegarde professionnelle (rétention, traçabilité, restauration)

### 2. Principes retenus

- **Simplicité et transparence** : utilisation de `rsync` (facile à comprendre, à auditer et à restaurer)
- **Cohérence avec le chiffrement** : toutes les sauvegardes sont stockées sur le volume LUKS
- **Échec propre** : le script doit détecter si le volume n’est pas monté et s’arrêter clairement
- **Séparation des rôles** : un script pour la G4, un mécanisme distinct pour OPNsense
- **Évolutivité** : la solution doit pouvoir intégrer plus tard une rotation et des notifications

### 3. Prérequis

| Élément | Statut attendu | Commentaire |
|---------|----------------|-----------|
| Volume LUKS `backup_crypt` | Créé et fonctionnel | Voir fiche 35 |
| Point de montage `/mnt/backup_g4` | Configuré (crypttab + fstab) | Montage semi-automatique |
| Structure de dossiers | Présente | `G4_Rocky/`, `OPNsense/`, `Archives/`, `Scripts/` |
| Outil `rsync` | Installé | Déjà présent sur la G4 |
| Espace disponible | Suffisant | ~938 Go utilisables |

Le volume doit être **déverrouillé et monté** pour que les backups puissent s’exécuter.

---

### 4. Script de backup de la G4

Pourquoi cette étape : Un script unique, testable et journalisé permet de standardiser la sauvegarde de la G4, de détecter clairement les échecs (volume non monté, erreur rsync) et de préparer une automatisation fiable via cron. Il constitue le cœur de la politique de sauvegarde locale.
#### 4.1 Création du script

```bash
sudo mkdir -p /mnt/backup_g4/Scripts
sudo nano /mnt/backup_g4/Scripts/backup_g4.sh
```

Contenu du script :

```bash
#!/bin/bash

# ============================================================
# Script de backup de la G4 vers le volume chiffré
# Projet : Bastion Godmode / Horus AIS
# Date   : 17 août 2026
# ============================================================

BACKUP_ROOT="/mnt/backup_g4"
BACKUP_DIR="${BACKUP_ROOT}/G4_Rocky"
LOG_FILE="${BACKUP_ROOT}/Scripts/backup_g4.log"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
TARGET="${BACKUP_DIR}/G4_Rocky_${DATE}"

if ! mountpoint -q "$BACKUP_ROOT"; then
    echo "[$(date)] ERREUR : Le volume $BACKUP_ROOT n'est pas monté." | tee -a "$LOG_FILE"
    exit 1
fi

mkdir -p "$TARGET"

echo "[$(date)] Début du backup de la G4 vers $TARGET" | tee -a "$LOG_FILE"

rsync -aAXv --delete \
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

#### 4.2 Points importants

- Vérification `mountpoint` : le script s’arrête si le volume n’est pas monté
- Code 23 accepté (attributs non supportés)
- Log centralisé dans `/mnt/backup_g4/Scripts/backup_g4.log`
- Dossier horodaté pour chaque exécution

#### 4.3 Rendre exécutable et tester

```bash
sudo chmod +x /mnt/backup_g4/Scripts/backup_g4.sh
sudo /mnt/backup_g4/Scripts/backup_g4.sh
sudo tail -n 30 /mnt/backup_g4/Scripts/backup_g4.log
ls -la /mnt/backup_g4/G4_Rocky/
```

![[Pasted image 20260817073321.png]]

**Résultat :** Code 0 – 19 Go – dossier `G4_Rocky_2026-08-17_21-00-53`

---

### 5. Planification cron
Pourquoi cette étape : L’automatisation via cron garantit l’exécution régulière du backup sans intervention manuelle. Le créneau de 03h00 (heure creuse) limite l’impact sur les performances.

```bash
sudo EDITOR=nano crontab -e
```

Ligne :

```bash
0 3 * * * /mnt/backup_g4/Scripts/backup_g4.sh
```

Backup quotidien à 03h00.

---

### 6. Automatisation du backup OPNsense
Pourquoi cette étape : Les configurations OPNsense représentent le cœur du périmètre de sécurité. Les sauvegarder automatiquement sur le volume chiffré de la G4 assure une restauration rapide et cohérente, tout en respectant le principe de centralisation des backups.
#### 6.1 Principe

Utilisation du plugin `os-sftp-backup` pour pousser les configurations `.xml` vers la G4.

#### 6.2 Utilisateur dédié
Pourquoi un utilisateur dédié : Appliquer le principe du moindre privilège. En cas de compromission de la clé, l’impact reste limité au dossier de backup.

```bash
sudo useradd -r -m -d /home/backupopn -s /sbin/nologin -c "OPNsense Backup User" backupopn
sudo chown -R backupopn:backupopn /mnt/backup_g4/OPNsense
sudo chmod 750 /mnt/backup_g4/OPNsense
```

![[Pasted image 20260817075440.png]]

#### 6.3 Clé SSH
Pourquoi une clé dédiée : Plus sûre et plus adaptée à l’automatisation qu’un mot de passe. Facilite la révocation.
```bash
sudo -u backupopn ssh-keygen -t ed25519 -f /home/backupopn/.ssh/id_ed25519 -N "" -C "opnsense-backup"
sudo cp /home/backupopn/.ssh/id_ed25519.pub /home/backupopn/.ssh/authorized_keys
sudo chown -R backupopn:backupopn /home/backupopn/.ssh
sudo chmod 700 /home/backupopn/.ssh
sudo chmod 600 /home/backupopn/.ssh/authorized_keys
sudo chmod 600 /home/backupopn/.ssh/id_ed25519
```

#### 6.4 Configuration OPNsense

- Installation du plugin `os-sftp-backup`
- URL finale : `sftp://backupopn@10.0.10.11/OPNsense`
- Clé privée complète collée
- Backup Count : 30

![[Pasted image 20260817085237.png]]

#### 6.5 Problèmes rencontrés et corrections

**Erreur initiale :** `Permission denied (publickey...)`  
**Cause :** utilisateur `backupopn` non autorisé dans le durcissement SSH (fiche 28).  
**Correction :** ajout dans les utilisateurs autorisés + `systemctl reload sshd`.

**Erreurs suivantes :** chemin relatif + permissions  
**Corrections :**

```bash
sudo ln -s /mnt/backup_g4/OPNsense /home/backupopn/OPNsense
sudo chown -h backupopn:backupopn /home/backupopn/OPNsense
sudo chmod 755 /mnt
sudo chmod 755 /mnt/backup_g4
sudo chown backupopn:backupopn /mnt/backup_g4/OPNsense
sudo chmod 750 /mnt/backup_g4/OPNsense
```

**Résultat final :**  
`Backup successful` – fichier `config-1787004379.7429.xml` (165 Ko) bien reçu.

![[Pasted image 20260817084404.png]]  
![[Pasted image 20260817084726.png]]
Le remote backup a nécessité plusieurs corrections (autorisation SSH, chemin relatif, permissions). La version finale fonctionnelle utilise :

- Utilisateur backupopn
- URL : sftp://backupopn@10.0.10.11/OPNsense
- Lien symbolique + permissions ajustées

**Résultat :** fichier config-1787004379.7429.xml correctement reçu.
#### 6.6 Fréquence

Par défaut : **1 fois par jour** (vers 1h–2h).  
Un nouveau fichier n’est poussé qu’en cas de changement de configuration.

---

### 7. Points de vigilance

| Point | Détail | Risque / Action |
|-------|--------|-----------------|
| Volume non monté | Script G4 et remote OPNsense échouent | Surveiller + alerte future |
| Utilisateur `backupopn` | Doit rester dans les utilisateurs SSH autorisés | Ne pas l’oublier lors des futurs durcissements |
| Lien symbolique | Obligatoire pour le plugin SFTP | À conserver |
| Permissions | `/mnt` et `/mnt/backup_g4` en 755 | Toute restriction excessive casse le backup |
| Rotation absente | Accumulation des dossiers | À mettre en place |
| Pas d’alerte email | Échecs silencieux | Évolution future |
| Clé privée | Stockée dans OPNsense | Protéger / révoquer si besoin |

---

### 8. Évolutions futures

1. **Rotation des sauvegardes** (priorité)  
2. **Alertes email** sur échec  
3. Shell restreint pour `backupopn`  
4. Chiffrement des configs OPNsense  
5. Externalisation  
6. Fiche de reprise post-crash

---

**Fin de la fiche 36**

Le socle d’automatisation des backups (G4 + OPNsense) est opérationnel.

---

