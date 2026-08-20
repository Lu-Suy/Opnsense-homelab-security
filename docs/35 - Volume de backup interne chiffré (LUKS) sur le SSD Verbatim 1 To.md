**35 – Volume de backup interne chiffré (LUKS) sur le SSD Verbatim 1 To**

**Projet :** Bastion Godmode / Horus AIS  
**Machine :** G4 (Rocky Linux 10)  
**Date :** 17 août 2026  

Il y a un disque de 1 To exploitable dans la G4.  
Décision : chiffrer tout le disque `/dev/sda` avec LUKS.

---

### Introduction – Pourquoi un volume de backup chiffré ?

Dans une architecture de défense, le chiffrement du système principal (LUKS) n’a de sens que si les sauvegardes le sont également.  

Un disque système chiffré protège les données au repos. Mais si les backups sont stockés en clair sur un autre support, une grande partie de cette protection est annulée : un attaquant qui obtient accès au volume de sauvegarde récupère l’ensemble des données sensibles (configurations, clés, logs, données applicatives, etc.).

**Objectifs professionnels de cette mise en place :**

1. **Cohérence de la protection des données**  
   Aligner le niveau de sécurité des sauvegardes sur celui du système de production. Le chiffrement ne doit pas s’arrêter à la machine principale.

2. **Résilience et continuité**  
   Disposer d’un volume de backup local, performant et chiffré, permettant une restauration rapide en cas d’incident (corruption, erreur humaine, compromission, panne matérielle).

3. **Défense en profondeur**  
   Appliquer le principe de moindre privilège et de segmentation aussi aux données de sauvegarde. Même en cas d’accès physique au serveur, les backups restent protégés par une passphrase distincte.

4. **Posture professionnelle / portfolio**  
   Dans un contexte orienté data center et entretien technique, la capacité à concevoir et documenter une chaîne de sauvegarde chiffrée (système + backups) démontre une approche mature de la protection de l’information.

5. **Préparation à l’automatisation**  
   Ce volume servira de cible pour les futures sauvegardes automatisées de la G4, des configurations OPNsense, et potentiellement d’autres composants de l’infrastructure.

Le choix de LUKS sur un disque interne dédié permet de conserver les backups à proximité immédiate de la machine (performances, simplicité de restauration) tout en maintenant un niveau de confidentialité élevé.

---

### Étape 1 – Démonter la partition

```bash
sudo umount /mnt/sda_ntfs
```

Vérification :

```bash
mount | grep sda
```

### Étape 2 – Effacer la table de partitions actuelle

```bash
sudo wipefs -a /dev/sda
sudo sgdisk --zap-all /dev/sda
sudo fdisk -l /dev/sda
```

Le disque ne contient plus aucune partition.

### Étape 3 – Créer une nouvelle table de partitions + une grande partition

```bash
sudo fdisk /dev/sda
```

Commandes utilisées dans `fdisk` :

- `g` → créer une nouvelle table de partitions GPT  
- `n` → créer une nouvelle partition (valeurs par défaut = tout le disque)  
- `w` → écrire les modifications et quitter  

Vérification :

```bash
sudo fdisk -l /dev/sda
```

![[Pasted image 20260817052628.png]]

Résultat :  
- `/dev/sda1` → 953,9 Go (Linux filesystem)

### Étape 4 – Chiffrement LUKS

```bash
sudo cryptsetup luksFormat /dev/sda1
```

- Confirmer avec `YES`  
- Définir une passphrase forte et la stocker dans le gestionnaire de mots de passe  

### Étape 5 – Ouvrir le volume chiffré

```bash
sudo cryptsetup open /dev/sda1 backup_crypt
ls -l /dev/mapper/backup_crypt
```

![[Pasted image 20260817052842.png]]

Le volume `/dev/mapper/backup_crypt` est prêt.

### Étape 6 – Création du système de fichiers

```bash
sudo mkfs.ext4 -L backup_g4 /dev/mapper/backup_crypt
```

Effet de la commande :
- `-L backup_g4` : attribue un label clair  
- Crée un système de fichiers ext4 propre sur le volume chiffré  

### Étape 7 – Montage du volume

```bash
sudo mkdir -p /mnt/backup_g4
sudo mount /dev/mapper/backup_crypt /mnt/backup_g4
```

Vérification :

```bash
df -h /mnt/backup_g4
ls -la /mnt/backup_g4
```

![[Pasted image 20260817052956.png]]

- Taille utilisable : **938 Go**  
- Système de fichiers propre (présence de `lost+found`)

### Étape 8 – Création de la structure de dossiers

```bash
sudo mkdir -p /mnt/backup_g4/{G4_Rocky,OPNsense,Prodesk,Archives,Scripts}
sudo chown -R root:root /mnt/backup_g4
sudo chmod -R 750 /mnt/backup_g4
```

Explication de la structure :
- `G4_Rocky` → backups de la machine actuelle  
- `OPNsense` → configurations firewall  
- `Prodesk` → anciennes sauvegardes  
- `Archives` → sauvegardes anciennes / ponctuelles  
- `Scripts` → scripts d’automatisation  
- Permissions 750 : root en écriture, groupe root en lecture  

Vérification :

```bash
ls -la /mnt/backup_g4
```

![[Pasted image 20260817053158.png]]

### 9. Finalisation du montage permanent

#### 9.1 Récupérer l’UUID de la partition LUKS

```bash
sudo cryptsetup luksUUID /dev/sda1
```

UUID obtenu :  
`2b5e7368-42af-4e7a-a4d7-749409974f0a`

#### 9.2 Configuration de `/etc/crypttab`

```bash
echo "backup_crypt UUID=2b5e7368-42af-4e7a-a4d7-749409974f0a none luks" | sudo tee -a /etc/crypttab
```

Effet :
- `backup_crypt` → nom du device mapper  
- `UUID=...` → identification de la partition  
- `none` → utilisation d’une passphrase (pas de fichier de clé)  
- `luks` → type de volume  

Vérification :

```bash
cat /etc/crypttab
```

![[Pasted image 20260817054119.png]]

#### 9.3 Configuration de `/etc/fstab`

```bash
echo "/dev/mapper/backup_crypt  /mnt/backup_g4  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab
```

Explication des options :
- `/dev/mapper/backup_crypt` → volume une fois ouvert  
- `/mnt/backup_g4` → point de montage  
- `ext4` → type de système de fichiers  
- `defaults,noatime` → options de montage  
- `0 2` → pas de dump, fsck en priorité 2  

Vérification :

```bash
tail -n 5 /etc/fstab
```

![[Pasted image 20260817054259.png]]

#### 9.4 Test du montage (sans redémarrer) – Version Rocky Linux

```bash
sudo umount /mnt/backup_g4
sudo cryptsetup close backup_crypt

sudo cryptsetup open /dev/sda1 backup_crypt
sudo systemctl daemon-reload
sudo mount /mnt/backup_g4

df -h /mnt/backup_g4
ls -la /mnt/backup_g4
```

![[Pasted image 20260817055110.png]]

Le volume se monte correctement. Les dossiers `Archives`, `G4_Rocky`, `OPNsense`, `Prodesk` et `Scripts` sont présents.

---

### Point de vigilance – Déverrouillage du volume de backup

Le volume de backup est chiffré avec LUKS et configuré pour un montage permanent via `/etc/crypttab` + `/etc/fstab`.

**Comportement actuel :**
- Le **montage** est automatique une fois le volume déverrouillé.
- Le **déverrouillage** n’est **pas** automatique à distance.
- Après un redémarrage, il est nécessaire de saisir manuellement la passphrase du volume de backup, sinon celui-ci reste inaccessible.

**Risque principal :**  
Oublier de déverrouiller le volume après un reboot → les futurs jobs de backup échoueront.

**Commandes de déverrouillage + montage à distance :**

```bash
sudo cryptsetup open /dev/sda1 backup_crypt
sudo systemctl daemon-reload
sudo mount /mnt/backup_g4
```

---

**Note :**  
Une fiche technique de reprise post-crash / redémarrage à distance sera rédigée ultérieurement. Elle regroupera dans l’ordre :
1. Déverrouillage du système principal (dracut-sshd)  
2. Déverrouillage + montage du volume de backup  
3. Vérification des services critiques (Docker, BunkerWeb, Wazuh, etc.)  
4. Points de contrôle  

---

**Document prêt pour intégration dans le dépôt portfolio**  
`docs/35 - Volume de backup interne chiffré (LUKS) sur le SSD Verbatim 1 To.md`
