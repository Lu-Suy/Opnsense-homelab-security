# 07b – Hardening du Bastion (détail pratique)

**Date originale** : 18 mai 2026  
**Machine** : Prodesk 600 Mini G2 (Debian 12) – IP management `10.0.10.y`  
**Version portfolio** : 15 août 2026 (sanitization)  
**Objectif** : Durcissement complet SSH + séparation des privilèges + protection anti-bruteforce

> **Note portfolio** :  
> Snapshot historique de l’ancien bastion (Prodesk).  
> Les usernames, IPs et noms de clés ont été généralisés.  
> Contenu technique et commandes conservés intégralement.

---

## 1. Séparation des utilisateurs (bonne pratique)

- `limited-user` → Utilisateur quotidien **limité** (pas de sudo)
- `admin-user` → Utilisateur **admin full sudo** (NOPASSWD)

### Commandes utilisées

```bash
# En root
adduser admin-user
usermod -aG sudo admin-user
echo "admin-user ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/admin-user
chmod 0440 /etc/sudoers.d/admin-user
```

## 2. Durcissement SSH (clé uniquement)

**Fichier** : `/etc/ssh/sshd_config` (partie durcie)

```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers limited-user@10.0.0.z admin-user@10.0.0.z
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
LoginGraceTime 20
Protocol 2
```

**Clé utilisée** : `id_ed25519` (ED25519 avec passphrase)

**Commande de test** (depuis la workstation admin) :

```powershell
ssh -i $HOME\.ssh\id_ed25519 admin-user@10.0.10.y
```

## 3. Configuration sudo

- `admin-user` → droits sudo sans mot de passe
- Fichier : `/etc/sudoers.d/admin-user`

> **Note** : L’option `NOPASSWD:ALL` pour l’utilisateur admin est pratique en environnement lab / homelab.  
> En production stricte, il est généralement préférable d’exiger le mot de passe même pour les commandes sudo (principe de moindre privilège renforcé).

## 4. Mise à jour système (patch CVE Dirty Frag / Fragnesia)

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

Kernel actuel à la date du document : `6.1.0-48-amd64` (18 mai 2026)

## 5. Fail2ban (anti-bruteforce SSH)

**Installation** :

```bash
sudo apt install fail2ban python3-systemd -y
```

**Configuration** : `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
bantime  = 1d
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 10.0.0.z 10.0.10.0/24 10.0.0.0/8

[sshd]
enabled  = true
backend  = systemd
port     = 22
filter   = sshd
maxretry = 4
bantime  = 2d
```

**Statut** : `sudo systemctl status fail2ban` → **active (running)**

**Commandes utiles** :

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd banip <IP>   # ban manuel
```

## Règles Firewall OPNsense associées (OPT1)

- Allow SSH : workstation-admin → Bastion_Manager (`10.0.10.y:22`)
- Block SSH vers Bastion_Web_Service (`10.0.10.x:22`)
- Block SSH vers Bastion_Manager depuis ailleurs

---

**Dernière mise à jour** : 18 mai 2026 – Fail2ban actif + séparation utilisateurs terminée.

**Prochaines étapes proposées à l’époque** :
- Google Authenticator (2FA) sur SSH
- Nettoyage général (fichier 04/05)
- BunkerWeb multisite + domaines

---

### Monitorer les tentatives SSH de deux façons :

#### 1. **La meilleure et la plus complète** (recommandée) :

```bash
# Voir les dernières tentatives de connexion SSH
sudo journalctl -u ssh -n 100 --no-pager

# Voir seulement les échecs / refus
sudo journalctl -u ssh | grep -E "Failed|Invalid|refused|preauth"

# En temps réel (comme un tail -f)
sudo journalctl -u ssh -f
```

#### 2. **Le fichier auth.log** (toujours utile) :

```bash
# Voir le fichier traditionnel
sudo tail -n 100 /var/log/auth.log | grep ssh

# En temps réel
sudo tail -f /var/log/auth.log | grep ssh
```

---

**Verdict** :
- Sur Debian 12 → `journalctl -u ssh` est **le plus fiable** (logs systemd).
- `auth.log` existe encore (grâce à rsyslog), mais il est moins complet que le journal.

Fail2ban lui-même lit déjà ces logs en arrière-plan (c’est pour ça qu’il fonctionne).

---

**Document historique – Hardening pratique de l’ancien bastion (Prodesk)**  
*Version portfolio sanitisée – 15 août 2026*
