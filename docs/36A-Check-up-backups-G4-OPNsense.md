
---

**36A – Check-up opérationnel des backups (G4 + OPNsense)**

**Projet :** Bastion Godmode / Horus AIS  
**Date :** 20 août 2026  
**Machine concernée :** G4 (Rocky Linux 10 – 10.0.10.11 sur OPT1) + OPNsense  
**Référence :** Fiche 36 (Automatisation des backups) + Fiche 35 (Volume LUKS)  
**Destination :** Volume chiffré `/mnt/backup_g4`

### 1. Objectif

Valider en conditions réelles que l’automatisation mise en place le 17 août 2026 fonctionne correctement, de façon fiable et sécurisée :

- Le volume LUKS `backup_crypt` est monté et accessible
- Les backups quotidiens de la G4 (rsync) sont présents, complets et datés
- Les configurations OPNsense arrivent bien via le plugin `os-sftp-backup`
- Permissions, utilisateur dédié, SELinux et durcissement SSH restent conformes
- Détecter les risques avant d’attaquer la rotation/rétention (fiche 37) et les alertes

Ce check-up s’inscrit dans l’esprit « pro / data-center » (traçabilité, moindre privilège, non-régression) et s’inspire de la méthodologie de documentation type Eric Dupaud.

**Note importante sur le schéma joint :**  
Le schéma réseau fourni montre encore l’ancien bastion Prodesk 600 G2 Mini (i3). Depuis la migration, le bastion principal est bien la **G4 Rocky Linux 10** à l’adresse **10.0.10.11** sur OPT1 (10.0.10.0/24). Le Prodesk peut éventuellement devenir honeypot plus tard.

### 2. Prérequis rappelés (état théorique au 17/08)

| Élément | Statut attendu | Commentaire |
|---------|----------------|-------------|
| Volume LUKS `backup_crypt` | Créé + monté | `/mnt/backup_g4` (ext4, label `backup_g4`) |
| Options crypttab | `noauto,nofail` | Pour éviter le prompt multi-LUKS au boot (voir fiche 37) |
| Script | `/mnt/backup_g4/Scripts/backup_g4.sh` | Exécutable, log local |
| Cron | `0 3 * * *` | Tous les jours à 03h00 |
| Utilisateur | `backupopn` (nologin) | Clé ed25519, dossier `/home/backupopn/OPNsense` → symlink |
| Plugin OPNsense | `os-sftp-backup` | URL `sftp://backupopn@10.0.10.11/OPNsense`, Backup Count 30 |
| Espace | ~938 Go utilisables | Attention à l’accumulation sans rotation |

### 3. Checklist de vérification (à exécuter sur la G4)

Exécute les blocs ci-dessous **un par un** (en root ou avec sudo) et **colle-moi les sorties**. On analysera ensemble et on complétera la section « Résultats du 20/08 ».

#### 3.1 Volume LUKS, montage et espace (critique)

```bash
# 1. Le point de montage est-il actif ?
mountpoint -q /mnt/backup_g4 && echo "OK : volume monté" || echo "ERREUR : volume NON monté"

# 2. Détails du montage
findmnt /mnt/backup_g4

# 3. Statut du device mapper LUKS
sudo cryptsetup status backup_crypt

# 4. Espace disque + inodes (très important sans rotation)
df -hT /mnt/backup_g4
df -i /mnt/backup_g4

# 5. Vue hardware rapide
lsblk -f | grep -E 'sda|backup|crypt'
```

**Pourquoi / Effet dans notre environnement :**  
Sur Rocky 10 avec crypttab en `noauto,nofail` (correctif multi-LUKS de la fiche 37), le volume peut ne pas être monté après un reboot. Cette série confirme qu’il est déverrouillé et accessible. `df -h` montre l’espace libre critique : à ~19 Go par backup complet, sans rotation on saturerait le Verbatim 1 To relativement vite.
![[Pasted image 20260820140123.png]]

#### 3.2 Structure, permissions et utilisateur backupopn

```bash
# Structure globale
ls -la /mnt/backup_g4/

# Dossiers critiques
ls -la /mnt/backup_g4/Scripts/
ls -la /mnt/backup_g4/G4_Rocky/ | head -n 8
ls -la /mnt/backup_g4/OPNsense/ | tail -n 10

# Symlink (obligatoire pour le plugin SFTP)
namei -l /home/backupopn/OPNsense

# Utilisateur et home
id backupopn
getent passwd backupopn
ls -la /home/backupopn/
ls -la /home/backupopn/.ssh/

# Permissions du chemin SFTP (doivent rester ouvertes en 755 jusqu’au dossier OPNsense)
ls -ld /mnt /mnt/backup_g4 /mnt/backup_g4/OPNsense
```

**Pourquoi / Effet :**  
Valide la hiérarchie créée en fiche 36, les droits stricts (backupopn:backupopn 750), le symlink qui permet au plugin os-sftp-backup de trouver le chemin relatif, et que l’utilisateur n’a pas de shell interactif (principe du moindre privilège).

#### 3.3 Script, cron et logs G4

```bash
# Présence et droits du script
ls -l /mnt/backup_g4/Scripts/backup_g4.sh
file /mnt/backup_g4/Scripts/backup_g4.sh
head -n 15 /mnt/backup_g4/Scripts/backup_g4.sh

# Cron
sudo crontab -l | grep -E 'backup|3 \*'
sudo systemctl status crond --no-pager -l | head -n 12

# Logs récents
sudo tail -n 50 /mnt/backup_g4/Scripts/backup_g4.log

# Recherche des derniers succès / échecs
grep -E 'Début du backup|terminé avec succès|ERREUR' /mnt/backup_g4/Scripts/backup_g4.log | tail -n 15
```

**Pourquoi / Effet :**  
Confirme que le script est exécutable, que cron l’appelle bien à 03h00, et que les dernières exécutions ont réussi (code 0 ou 23 accepté pour les xattrs SELinux). Les logs sont le seul endroit où un échec « volume non monté » apparaît clairement.


### Synthèse rapide

|Point|Statut|Commentaire|
|---|---|---|
|Structure dossiers|**OK**|Archives / G4_Rocky / OPNsense / Scripts / Prodesk|
|Backups G4 quotidiens|**OK**|4 dossiers présents (17 → 20 août)|
|Cron 03h00|**OK**|Exactement la bonne ligne|
|Logs de succès|**OK**|Tous les backups en **code 0**|
|Configs OPNsense|**OK**|3 fichiers XML reçus (17, 19, 20 août)|
|Symlink + permissions SFTP|**OK**|Parfait|
|Script exécutable|**OK**|Présent et correct|

### Points d’attention (pas bloquants mais à traiter)

1. **Shell de backupopn = /bin/bash** Dans la fiche 36 on avait prévu /sbin/nologin (moindre privilège). Actuellement il a un vrai shell. Ce n’est pas critique (il n’a pas de sudo et le home est bien isolé), mais on va le corriger proprement.
2. **Log backup_g4.log déjà à 128 Mo** Le script loggue _tout_ le détail de rsync (-v). C’est très verbeux. On gérera ça dans la rotation (fiche 37) en passant en mode plus silencieux + rotation du log.

Le reste est vraiment propre.

---

### Prochaines commandes (on avance)

Peux-tu me coller maintenant ces trois blocs (ils sont courts) :

**A. Tailles des backups G4 (très utile pour la future rotation)**

Bash

```
du -sh /mnt/backup_g4/G4_Rocky/*
du -sh /mnt/backup_g4/OPNsense/
du -sh /mnt/backup_g4/Scripts/
```

**B. Version rsync (alerte sécurité du 13 août)**

Bash

```
rsync --version
rpm -q rsync
```

**C. SELinux + shell (pour valider le point de moindre privilège)**

Bash

```
getenforce
ls -Zd /mnt/backup_g4 /mnt/backup_g4/OPNsense /mnt/backup_g4/Scripts
getent passwd backupopn
```

![[Pasted image 20260820142302.png]]

### Correctif rapide proposé (shell de backupopn)

Actuellement backupopn a un shell /bin/bash. On veut le passer en nologin (moindre privilège).

 On fera simplement :

Bash

```
sudo usermod -s /sbin/nologin backupopn
getent passwd backupopn   # pour vérifier
```
![[Pasted image 20260820142811.png]]

Ça n’impacte **pas** le backup SFTP (le plugin utilise la clé, pas de login interactif).


#### 3.4 Contenu des backups G4 (growth & intégrité basique)

```bash
# Taille globale et par dossier
du -sh /mnt/backup_g4/*
du -sh /mnt/backup_g4/G4_Rocky/* | sort -h | tail -n 10

# Les 5 derniers backups (date + taille)
ls -lth /mnt/backup_g4/G4_Rocky/ | head -n 6

# Test d’intégrité légère (fichier système non sensible)
LATEST=$(ls -t /mnt/backup_g4/G4_Rocky/ | head -1)
echo "Dernier dossier : $LATEST"
ls /mnt/backup_g4/G4_Rocky/$LATEST/etc/os-release
cat /mnt/backup_g4/G4_Rocky/$LATEST/etc/os-release | head -n 5
```

**Pourquoi / Effet :**  
Montre l’accumulation (sans rotation → croissance linéaire). Le test sur `os-release` prouve que les données sont lisibles et non corrompues.

#### 3.5 Côté OPNsense (fichiers reçus)

```bash
# Fichiers config reçus (triés par date)
ls -lth /mnt/backup_g4/OPNsense/ | head -n 15

# Nombre et taille totale
echo "Nombre de fichiers .xml :"
ls /mnt/backup_g4/OPNsense/*.xml 2>/dev/null | wc -l
du -sh /mnt/backup_g4/OPNsense/

# Aperçu d’un fichier récent
LATEST_XML=$(ls -t /mnt/backup_g4/OPNsense/*.xml 2>/dev/null | head -1)
echo "Dernier XML : $LATEST_XML"
head -c 300 "$LATEST_XML"
echo
```

**Pourquoi / Effet :**  
Vérifie que le plugin os-sftp-backup pousse bien les configs (Backup Count 30). Un XML lisible prouve que le chemin, les permissions et la clé fonctionnent.

**Côté GUI OPNsense :** System → Configuration → Backups ou plugin Remote/SFTP → vérifier le statut « Last successful backup ».

#### 3.6 Sécurité du process (SELinux + SSH hardening + moindre privilège)

```bash
# SELinux
getenforce
ls -Zd /mnt/backup_g4 /mnt/backup_g4/Scripts /mnt/backup_g4/OPNsense /mnt/backup_g4/G4_Rocky

# Derniers denials éventuels
sudo ausearch -m avc -ts recent 2>/dev/null | grep -E 'backup|rsync|sftp|backupopn' | tail -n 15 || echo "Aucun AVC récent lié aux backups"

# Durcissement SSH (fiche 28) – backupopn doit être autorisé
sudo grep -E 'AllowUsers|Match User|backupopn' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/* 2>/dev/null

# Test nologin
sudo -u backupopn -s 2>&1 | cat

# Fail2ban (pour info)
sudo fail2ban-client status sshd 2>/dev/null | head -n 20 || echo "Fail2ban non actif ou pas de jail sshd"
```

**Pourquoi / Effet :**  
Rocky 10 est Enforcing par défaut. Des denials SELinux peuvent bloquer silencieusement rsync ou sftp-server. On vérifie aussi que le durcissement SSH (AllowUsers) n’a pas drifté et que backupopn reste limité au nologin + SFTP.

#### 3.7 Version rsync (alerte sécurité du 13 août 2026)

```bash
rsync --version
rpm -q rsync
dnf check-update rsync 2>/dev/null || true
```

**Contexte (vérifié online le 20/08/2026) :**  
rsync **3.5.0** est sorti le **13 août 2026**. Il corrige **33 vulnérabilités** (1 Critical CVE-2026-53791 + 17 High + 15 Medium), principalement sur le daemon, le path handling, les symlinks et les races TOCTOU.

Dans notre cas :
- On utilise rsync **en local** (pas de daemon rsyncd, pas de PROXY protocol) → impact fortement réduit.
- Cependant, les corrections path handling / symlink / xattrs restent pertinentes pour un rsync root avec nos flags `-aAXv --delete`.
- **Bonne pratique pro** : vérifier la version et planifier l’update si < 3.5.0 (surtout après le dnf update du 17/08).

Si la version est < 3.5.0 → on planifie un `dnf update rsync` + re-test du script.

### 4. Test de restauration minimal (non destructif)

```bash
# Créer un dossier de test
mkdir -p /tmp/restore_test_36A
LATEST=$(ls -t /mnt/backup_g4/G4_Rocky/ | head -1)

# Restaurer uniquement un fichier non critique
sudo rsync -a /mnt/backup_g4/G4_Rocky/$LATEST/etc/hostname /tmp/restore_test_36A/
cat /tmp/restore_test_36A/hostname

# Nettoyage
rm -rf /tmp/restore_test_36A
```

**Effet attendu :**  
Le hostname de la G4 s’affiche correctement → preuve que le backup est lisible et restaurable.

### 5. Tableau de synthèse (à remplir ensemble)

| Point de contrôle                  | Statut (OK / KO / À surveiller) | Commentaire / Action |
|------------------------------------|---------------------------------|----------------------|
| Volume monté                       |                                 |                      |
| Espace libre                       |                                 |                      |
| Script + cron                      |                                 |                      |
| Dernier backup G4 < 24-48h         |                                 |                      |
| Fichiers OPNsense présents ≤30     |                                 |                      |
| Permissions + nologin + symlink    |                                 |                      |
| SELinux / pas d’AVC bloquant       |                                 |                      |
| AllowUsers backupopn               |                                 |                      |
| Version rsync ≥ 3.5.0              |                                 |                      |
| Test restore minimal OK            |                                 |                      |

### 6. Points de vigilance (rappel)

1. **Volume non monté** (noauto,nofail) → échec silencieux de tout le système de backup.
2. **Accumulation** sans rotation → saturation progressive du volume 1 To (~19 Go/jour).
3. **Pas d’alertes email** encore → les échecs restent silencieux pour l’instant.
4. **Clé privée dans OPNsense** → à protéger / envisager rotation.
5. **SELinux** après updates kernel ou restorecon manqué.
6. **rsync < 3.5.0** → à traiter rapidement même si impact local limité.

### 7. Actions immédiates selon résultats

- Si volume non monté → déverrouiller manuellement + investiguer fstab/crypttab.
- Si permissions cassées → reprendre les commandes de la fiche 36 (section 6.5).
- Si rsync ancien → planifier update + re-test.
- Si growth trop rapide → accélérer la fiche 37 (rotation + rétention).

### 8. Suite logique

Une fois ce check-up validé :

1. **Fiche 37** (déjà commencée) : rotation + rétention + test restore plus complet + correctif multi-LUKS.
2. Alertes email (Fail2ban / Wazuh / OPNsense / script backup → alertes@horus-ais.com).
3. Éventuellement monitoring Wazuh (file integrity sur le log + script, ou règle custom sur échec).

---

**Fin de la fiche 36A – Check-up**

