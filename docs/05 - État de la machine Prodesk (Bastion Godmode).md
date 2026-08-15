**Date** : 18 mai 2026  
**Machine** : Prodesk 600 Mini G2  
**Objectif** : Garder une vue d’ensemble claire de l’état hardware, software et sécurité de la machine principale.

## 1. Informations générales

- **OS** : Debian 12 Bookworm (kernel 6.1.0-48-amd64 – patché Dirty Frag / Fragnesia)
- **CPU** : AMD Ryzen 7 4800H
- **RAM** : 16 Go (15 GiB disponible)
- **Disque système** : SSD 1 To NVMe (/dev/sda1) → ~2.9 Go utilisés après nettoyage
- **Hostname** : bastion-godmode
- **Utilisateur principal limité** : `doo`
- **Utilisateur admin** : `hyper_doo` (full sudo sans mot de passe)

## 2. Configuration Réseau

- **IP Web / Services** : 10.0.10.10/24 (sur OPT1)
- **IP Manager / SSH** : 10.0.10.11/24 (sur OPT1)
- **Gateway** : 10.0.10.1 (OPNsense OPT1)
- **DNS** : 10.0.0.1 (OPNsense)

## 3. Services actifs

- **BunkerWeb 1.6.9** (Docker) → Fonctionnel avec page statique personnalisée (`http://10.0.10.10`)
- **Fail2ban** → Actif avec jail SSH (backend systemd)
- **SSH** → Durci (clé ED25519 uniquement)
- **Docker** → Actif (réseaux bw-universe + bw-services)

## 4. Durcissement appliqué

### Utilisateurs & sudo
- `d**` → Utilisateur limité (aucun droit sudo)
- `hyper_d**` → Admin full sudo (NOPASSWD)

### SSH (/etc/ssh/sshd_config)
```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers d**@10.0.0.* hyper_d**@10.0.0.*
MaxAuthTries 3
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2
```

**Clé utilisée** : `id_ed25519` (ED25519 avec passphrase)

### Fail2ban
- Jail `sshd` active
- Ban après 4 échecs → 2 jours
- Ignore : Alphadeck (10.0.0.**) + réseaux locaux

## 5. Sauvegardes

- **Dernière sauvegarde complète** : 14 mai 2026
- **Emplacement** : /mnt/external/backup_prodesk_20260514/
- **Taille** : ~192 Mo
- **Contenu** : /etc/, /home/, /var/lib/docker/, etc.

## 6. État actuel & recommandations

**État** : Machine propre, durcie, à jour et fonctionnelle.  
**Prochaines actions prioritaires** :
- Nettoyage général + optimisation (fichier 04)
- BunkerWeb multisite + Let’s Encrypt (fichier 11)
- OpenVPN / WireGuard (fichier 12)
- Wazuh SIEM

**Dernière mise à jour** : 18 mai 2026 – Full-upgrade terminé, Fail2ban actif, SSH durci, BunkerWeb fonctionnel.

---
