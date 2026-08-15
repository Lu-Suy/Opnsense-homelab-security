# 05 – État de la machine Prodesk (Bastion )

**Date originale** : 18 mai 2026  
**Machine** : Prodesk 600 Mini G2  
**Version portfolio** : 15 août 2026 (sanitization)  
**Objectif** : Garder une vue d’ensemble claire de l’état hardware, software et sécurité de la machine principale (snapshot historique – phase avant migration G4).

> **Note portfolio** :  
> Les adresses IP, hostnames et usernames ont été généralisés.  
> Contenu technique conservé intégralement (document historique de mai 2026).

---

## 1. Informations générales

- **OS** : Debian 12 Bookworm (kernel 6.1.0-48-amd64 – patché Dirty Frag / Fragnesia)
- **CPU** : Intel i3-6100T
- **RAM** : 16 Go (15 GiB disponible)
- **Disque système** : SSD 1 To NVMe (/dev/sda1) → ~2.9 Go utilisés après nettoyage
- **Hostname** : bastion (généralisé)
- **Utilisateur principal limité** : `admin-user`
- **Utilisateur admin** : `admin-full` (full sudo sans mot de passe)

## 2. Configuration Réseau

- **IP Web / Services** : `10.0.10.x/24` (sur OPT1)
- **IP Manager / SSH** : `10.0.10.y/24` (sur OPT1)
- **Gateway** : `10.0.10.1` (OPNsense OPT1)
- **DNS** : `10.0.0.1` (OPNsense)

## 3. Services actifs

- **BunkerWeb 1.6.9** (Docker) → Fonctionnel avec page statique personnalisée (`http://10.0.10.x`)
- **Fail2ban** → Actif avec jail SSH (backend systemd)
- **SSH** → Durci (clé ED25519 uniquement)
- **Docker** → Actif (réseaux bw-universe + bw-services)

## 4. Durcissement appliqué

### Utilisateurs & sudo
- `admin-user` → Utilisateur limité (aucun droit sudo)
- `admin-full` → Admin full sudo (NOPASSWD)

### SSH (/etc/ssh/sshd_config)
```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin-user@10.0.0.* admin-full@10.0.0.*
MaxAuthTries 3
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2
```

**Clé utilisée** : `id_ed25519` (ED25519 avec passphrase)

### Fail2ban
- Jail `sshd` active
- Ban après 4 échecs → 2 jours
- Ignore : workstation-admin (`10.0.0.z`) + réseaux locaux

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

**Document historique – Snapshot 18 mai 2026 (phase Prodesk)**  
*Version portfolio sanitisée – 15 août 2026*
