# 27 – Déverrouillage distant LUKS via dracut-sshd (G4)

**Projet :** Horus AIS – bastion  
**Date :** 10 août 2026 (sanitisé portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Objectif :** Saisir la passphrase LUKS à distance (SSH dans l’initramfs) pendant le boot, sans écran branché en permanence.

> **Publication :** IP de management masquée en `10.0.10.y`. Clé nommée `id_ed25519`. En lab, utiliser les valeurs réelles du plan d’adressage.

---

## 1. Contexte et besoin

Le G4 est chiffré avec **LUKS** (volume racine).  
Au démarrage, le système demande la passphrase **avant** de monter les systèmes de fichiers.

**Problème :**

- sans écran + clavier → impossible d’entrer la passphrase ;
- la machine reste bloquée au prompt cryptsetup.

**Objectif :**  
ouvrir une session SSH **très tôt** (initramfs), depuis le poste d’administration, et déverrouiller à distance.

---

## 2. Choix de solution : dracut-sshd (plutôt que Dropbear)

### Tentative Dropbear

Dropbear est classique pour ce usage, mais sur Rocky 10 l’intégration manuelle s’est révélée fragile :

- module personnalisé difficile à caler au bon moment du boot ;
- réseau pas toujours disponible assez tôt ;
- moins de retours récents stables sur cette combinaison.

### Pourquoi dracut-sshd

| Critère | Dropbear (lab) | dracut-sshd (OpenSSH) |
|---------|----------------|------------------------|
| Maturité Rocky 10 | Moyenne | Meilleure |
| Stabilité observée | Capricieuse | Plus fiable |
| Paquet | Partiel / bricolage | Dépôts |
| Auth | Clé | OpenSSH + clé |
| Maintenance | Plus manuelle | Plus simple |

**Décision :** `dracut-sshd` — OpenSSH embarqué dans l’initramfs via un module dracut.

---

## 3. Prérequis déjà en place

- Rocky 10 installé **avec LUKS**
- Réseau management en régime normal : `10.0.10.y/24`
- Clé SSH `id_ed25519` déjà utilisée pour les comptes admin du système
- Interface : `eno1`
- Gateway : `10.0.10.1`

---

## 4. Installation de dracut-sshd

```bash
dnf search dracut-sshd
sudo dnf install -y dracut-sshd
```

Le paquet ajoute un module dracut (ex. `46sshd`) qui démarre un serveur OpenSSH **très tôt** dans l’initramfs.

```bash
rpm -q dracut-sshd
# exemple lab : dracut-sshd-0.7.1-3.el10_2.noarch
```

---

## 5. Authentification – clé SSH pour root (initramfs)

Pendant le boot distant, on se connecte en **root** dans l’environnement initramfs.  
Il faut donc autoriser la **clé publique** du poste d’admin pour root.

```bash
sudo mkdir -p /root/.ssh

# Une seule ligne : clé publique du poste d’administration
echo "ssh-ed25519 AAAA... commentaire-poste-admin" | sudo tee /root/.ssh/authorized_keys

sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
```

| Point | Pourquoi |
|-------|----------|
| Clé **publique** seulement | La privée ne quitte pas le poste admin |
| `700` / `600` | Sans ça, sshd refuse la clé |
| Compte **root** ici | Identité attendue dans l’environnement early-boot |

```bash
sudo ls -la /root/.ssh/
```

---

## 6. Réseau au boot (point critique)

Sans IP dans l’initramfs, pas de SSH distant — timeout depuis le poste admin.

### 6.1 Première tentative — insuffisante sous BLS

Ajout dans `/etc/default/grub` :

```bash
GRUB_CMDLINE_LINUX="... rd.neednet=1 ip=10.0.10.y::10.0.10.1:255.255.255.0:bastion:eno1:none"
```

Puis :

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Problème rencontré :**  
Rocky utilise **BLS** (Boot Loader Specification).  
Modifier seulement `/etc/default/grub` + `grub2-mkconfig` **ne propage pas toujours** les arguments vers les entrées de boot réelles.

Après reboot, `/proc/cmdline` **sans** `rd.neednet=1` ni `ip=...` → pas de réseau early → SSH impossible.

### 6.2 Correction qui a fonctionné : `grubby`

```bash
sudo grubby --update-kernel=ALL \
  --args="rd.neednet=1 ip=10.0.10.y::10.0.10.1:255.255.255.0:bastion:eno1:none"
```

| Paramètre | Rôle |
|-----------|------|
| `rd.neednet=1` | Force le réseau dans l’initramfs |
| `ip=client::gw:netmask:hostname:iface:none` | IP statique early-boot |
| client | `10.0.10.y` (management) |
| gw | `10.0.10.1` |
| hostname | `bastion` |
| iface | `eno1` |
| `none` | Pas d’autoconf DHCP |

Vérification :

```bash
grubby --info=ALL | grep args
```

Les `args=` de **toutes** les entrées kernel doivent contenir `rd.neednet=1` et `ip=...`.

---

## 7. Reconstruction de l’initramfs

```bash
sudo dracut -f
# ou, pour contrôler l’inclusion du module :
sudo dracut -f -v
```

Dans la sortie verbose, on doit voir notamment :

```text
*** Including module: sshd ***
```

Sans ce module dans l’image, pas de serveur SSH early-boot.

---

## 8. Test de fonctionnement

### 8.1 Reboot

```bash
sudo reboot
```

### 8.2 Connexion depuis le poste d’administration

Dès que le boot atteint l’initramfs (réseau up) :

```powershell
ssh -i $HOME\.ssh\id_ed25519 root@10.0.10.y
```

- Passphrase de la **clé privée** SSH : éventuelle (côté client) — normale.
- Pas de mot de passe **compte root** si seule la clé est autorisée.

### 8.3 Déverrouillage LUKS

Message typique une fois connecté :

```text
Welcome to the early boot SSH environment.
You may type
    systemd-tty-ask-password-agent
to unlock your disks.
initramfs-ssh:/root#
```

```bash
systemd-tty-ask-password-agent
```

Saisir la **passphrase LUKS**.  
Le shell early-boot se ferme ; le boot continue jusqu’au système normal.

---

## 9. Erreur / correction (runbook)

| Étape | Résultat |
|-------|----------|
| Install `dracut-sshd` | OK |
| Clé dans `/root/.ssh` | OK |
| `/etc/default/grub` + `grub2-mkconfig` seul | **Insuffisant** (BLS) |
| **`grubby --update-kernel=ALL --args=...`** | **Correction critique** |
| `dracut -f` + reboot | Succès |

**Cause racine :** arguments réseau absents du cmdline réel à cause de BLS.  
**Remède :** injecter via **`grubby`**, pas seulement via le fichier GRUB « classique ».

---

## 10. Sécurité

- Auth initramfs **par clé uniquement** (pas de password root early-boot)
- Clé privée uniquement sur le poste d’admin
- Daemon SSH early-boot limité à la phase de déverrouillage
- Après pivot root, le SSH « système » reprend sa config habituelle

**Suite logique (doc 28) :**  
`PasswordAuthentication no` sur le sshd du système monté, `AllowUsers`, etc.

---

## 11. État final

| Élément | Statut |
|---------|--------|
| `dracut-sshd` | OK |
| Clé root pour initramfs | OK |
| Réseau early-boot via `grubby` | OK |
| Initramfs reconstruit | OK |
| SSH pendant le boot | OK |
| Unlock LUKS distant | OK |

---

## 12. Suite

- Durcissement SSH (doc 28)
- Fail2ban (doc 29)
- Procédure recovery (live USB) si initramfs / grubby mal configuré
- WireGuard, Wazuh (docs ultérieures)

---

**Document validé en lab – version portfolio / runbook – 15 août 2026**
