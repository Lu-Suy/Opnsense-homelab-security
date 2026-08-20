
---

**34 – Premier backup complet de la G4 (Rocky Linux) vers disque externe**

**Projet :** Bastion Godmode / Horus AIS  
**Date :** 17 août 2026  
**Machine :** G4 (Rocky Linux 10 + LUKS)  
**Destination :** Disque externe `/dev/sdb2` (label BACK_Up)

### Introduction – Pourquoi un backup externe ?

Avant de mettre en place un volume de sauvegarde interne chiffré, il était nécessaire de réaliser un **premier backup complet** de la G4 vers un support externe.

Ce backup externe répond à plusieurs objectifs :

1. **Sortir du risque zéro**  
   Jusqu’à cette date, aucune sauvegarde complète de la G4 n’existait. Un incident aurait pu entraîner une perte totale.

2. **Sécuriser les opérations suivantes**  
   La création du volume de backup interne (chiffrement LUKS du SSD Verbatim) implique des opérations destructives sur `/dev/sda`. Un backup externe préalable permet de conserver une copie de secours.

3. **Externalisation temporaire**  
   Le disque externe sert de support de dépannage et d’externalisation rapide. Il ne constitue pas la solution définitive, mais offre une première couche de résilience hors de la machine.

4. **Posture professionnelle**  
   Toute opération sensible (repartitionnement, chiffrement) doit être précédée d’une sauvegarde validée.

---

### 1. Identification du disque externe

```bash
lsblk
```

Résultat pertinent :
- `sdb` → 931,5 Go (disque externe)
- `sdb1` → label `WD_Blue` (NTFS)
- `sdb2` → label `BACK_Up` (NTFS)

Vérification des systèmes de fichiers :

```bash
sudo blkid /dev/sdb1 /dev/sdb2
```

- `/dev/sdb1` : NTFS – `WD_Blue`
- `/dev/sdb2` : NTFS – `BACK_Up`

**Décision :** Utiliser uniquement la partition `sdb2` (`BACK_Up`) pour les sauvegardes.

---

### 2. Montage du disque externe

Création du point de montage et montage en lecture-écriture :

```bash
sudo mkdir -p /mnt/backup_externe
sudo mount -o rw,uid=0,gid=0,umask=000 /dev/sdb2 /mnt/backup_externe
```

Vérification :

```bash
mount | grep backup_externe
ls -la /mnt/backup_externe
```

Contenu initial de la partition :
- `OPNSENS_Back_Up/` (configurations OPNsense)
- `prodesk-debian-2026-08-05/` (ancienne sauvegarde Prodesk)
- Dossiers système Windows

---

### 3. Préparation du dossier de backup

```bash
sudo mkdir -p /mnt/backup_externe/G4_Rocky_$(date +%Y-%m-%d)
ls -la /mnt/backup_externe
```

Dossier créé : `G4_Rocky_2026-08-17`

---

### 4. Installation de rsync

```bash
sudo dnf install -y rsync
rpm -q rsync
```

---

### 5. Exécution du backup

Commande utilisée :

```bash
sudo rsync -aAXv --delete \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/home/*/.cache"} \
  / /mnt/backup_externe/G4_Rocky_2026-08-17/
```

**Particularité rencontrée :**  
De nombreuses erreurs du type :

```
rsync_xal_set: lsetxattr(... "security.selinux") failed: Operation not supported (95)
```

**Cause :**  
Le système source utilise SELinux. Le système de fichiers de destination (NTFS) ne supporte pas les attributs étendus SELinux.

**Impact :**  
Les fichiers sont correctement copiés. Seuls les labels SELinux ne peuvent pas être conservés. Comportement attendu et non bloquant.

---

### 6. Résultat du backup

```bash
echo $?
du -sh /mnt/backup_externe/G4_Rocky_2026-08-17
ls /mnt/backup_externe/G4_Rocky_2026-08-17
```

- Volume sauvegardé : **19 Go**
- Code de retour rsync : **23** (attributs SELinux non supportés)
- Structure présente : `bin`, `boot`, `etc`, `home`, `root`, `usr`, `var`, etc.

**Statut :** Backup valide (premier backup complet de la G4)

---

### 7. Sauvegarde complémentaire – Dossier Lan

Le dossier `Lan` présent sur le disque interne `sda` (NTFS) a également été copié sur le disque externe avant les opérations destructives :

```bash
sudo rsync -aAXv /mnt/sda_ntfs/Lan /mnt/backup_externe/
du -sh /mnt/backup_externe/Lan
```

Résultat : 5,1 Go copiés avec succès.

---

### 8. Points de vigilance

- Le format NTFS limite la conservation des attributs Linux (SELinux, ACL avancées, etc.).
- Ce backup externe constitue une solution de **dépannage / externalisation temporaire**, pas la solution définitive.
- La solution cible reste le volume interne chiffré LUKS (voir fiche 35).

---

### 9. Suite

- Mise en place du volume de backup interne chiffré (fiche 35)
- Automatisation ultérieure des sauvegardes
- Définition d’une politique de rétention

---

