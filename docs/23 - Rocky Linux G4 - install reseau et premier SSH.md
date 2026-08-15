# 23 – Rocky Linux G4 – install réseau et premier SSH

**Projet :** Horus AIS – bastion  
**Date :** 5 août 2026 (fusion portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (i5 8ᵉ génération)  
**Objectif :** Installer Rocky Linux 10 sur le G4, configurer le réseau et obtenir un premier accès SSH stable depuis le poste d’administration, en vue de remplacer l’ancien mini-PC comme bastion principal.

> **Publication :** les derniers octets des adresses du plan d’administration sont masqués (`x`, `y`). En lab, remplacer par les valeurs réelles du plan d’adressage.

---

## 1. Contexte

L’ancien bastion (mini-PC, Debian 12) arrivait en limite pour faire cohabiter confortablement WAF + tunnel + SIEM.  
Décision : basculer le rôle de **bastion principal** sur l’**EliteDesk 800 G4**.

| Machine | Rôle envisagé |
|---------|----------------|
| **EliteDesk 800 G4** | Bastion principal (Rocky Linux) |
| **Ancien mini-PC** | Honeypot / lab secondaire (plus tard) |

---

## 2. Backup avant migration

Sauvegarde des éléments critiques de l’ancien bastion sur disque externe :

- `/etc/cloudflared/`
- `/opt/bunkerweb/`
- Configurations SSH, utilisateurs, éventuel Fail2ban
- Documentation et scripts utiles à la reprise

Sans ce backup, la migration des services (tunnel, WAF) serait beaucoup plus risquée.

---

## 3. Installation Rocky Linux 10.2

| Paramètre | Choix |
|-----------|--------|
| ISO | `Rocky-10.2-x86_64-dvd1.iso` (install hors ligne) |
| Médium | Clé USB, **Rufus en mode DD Image** (important pour Anaconda) |
| Type | **Minimal Install** |
| Chiffrement | **LUKS** activé |
| Hostname | `bastion` |
| Utilisateur install | `doo` (admin à ce stade ; privilèges affinés plus tard) |

**Disques :**

- **NVMe ~250 Go** → système (Rocky + LUKS)
- **Disque 1 To** → non touché à l’install (data / backup)

### Point d’attention – médium d’install

En mode ISO « classique » (non DD), Anaconda peut échouer à lire le dépôt local (`failed to download repo.xml` / media not good) même avec le DVD complet.  
**Correction :** recréer la clé en **DD Image mode** dans Rufus, puis « auto-detect installation media ».

---

## 4. Configuration réseau (post-install)

### 4.1 Problème rencontré – IP perdues au reboot

**Symptôme :** les adresses saisies dans l’installateur graphique ne sont plus là après le premier boot ; ping / SSH impossibles de façon stable.

**Cause :** sous Rocky, **NetworkManager** reprend la main. La config « one-shot » de l’installeur n’est pas toujours persistée comme connexion NM active.

**Correction :** reconfigurer explicitement via `nmcli` et forcer le reconnexion.

```bash
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "10.0.10.x/24,10.0.10.y/24" \
  ipv4.gateway "10.0.10.1" \
  ipv4.dns "1.1.1.1 8.8.8.8" \
  connection.autoconnect yes

nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"
```

| Adresse (schéma public) | Rôle |
|-------------------------|------|
| `10.0.10.x/24` | Services (WAF, etc.) |
| `10.0.10.y/24` | Management / SSH |
| `10.0.10.1` | Gateway (pare-feu, interface OPT) |
| `1.1.1.1` / `8.8.8.8` | DNS **temporaire** pour permettre `dnf` |

### 4.2 Problème rencontré – pas de résolution de noms

**Symptôme :** `ping` IP OK parfois, mais `dnf install` échoue (résolution DNS).

**Cause :** DNS du plan local pas encore joignable / pas encore fiable au tout début ; sans DNS public temporaire, les dépôts ne résolvent pas.

**Correction :** DNS public le temps de l’amorçage (`1.1.1.1`, `8.8.8.8`), puis revenir plus tard au DNS du lab si besoin.

Vérification :

```bash
ping -c 3 1.1.1.1
ping -c 3 google.com
```

### 4.3 Ancienne route statique (ancien bastion)

Sur l’ancien hôte, une route du type « réseau LAN via une autre gateway » avait été ajoutée (contraintes firewall de l’époque).

Sur le G4 : **configuration propre** uniquement — gateway = interface OPT du pare-feu (`10.0.10.1`).  
Pas de route statique supplémentaire nécessaire pour un accès Internet + SSH management dans le plan actuel.

---

## 5. SSH – installation et premier accès

### 5.1 Installation du service

```bash
dnf install -y openssh-server
systemctl enable --now sshd
```

Ouvrir le port SSH dans **firewalld** si besoin (selon l’état par défaut de la Minimal Install).

### 5.2 Problème rencontré – host key changed

**Symptôme (depuis le poste Windows d’admin) :**

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
Host key verification failed.
```

**Cause :** l’IP de management `10.0.10.y` était déjà connue dans `known_hosts` pour **l’ancien** bastion. Le G4 est une **nouvelle** machine → nouvelles host keys. OpenSSH refuse de continuer (protection MITM).

**Ce qu’on ne change pas :** la **clé utilisateur** `id_ed25519` (toujours la même sur le poste d’admin).

| Type de clé | Où | Rôle | On change ? |
|-------------|-----|------|-------------|
| Clé **utilisateur** (`id_ed25519`) | Poste admin | Prouver *qui* se connecte | **Non** |
| Clé **hôte** (host key) | Serveur | Identifier *la machine* | Oui à chaque réinstall |

**Correction :**

```powershell
ssh-keygen -R 10.0.10.y
```

Puis :

```powershell
ssh -i $HOME\.ssh\id_ed25519 doo@10.0.10.y
```

Accepter la **nouvelle** empreinte une fois, puis s’authentifier (mot de passe `doo` à ce stade ; auth par clé uniquement plus tard).

---

## 6. État à la fin de cette étape

| Élément | Statut | Commentaire |
|---------|--------|-------------|
| Rocky Minimal + LUKS | OK | NVMe système |
| Hostname | OK | `bastion` |
| IP services + management | OK | schéma `x` / `y` |
| DNS (temp public) | OK | permet `dnf` |
| `sshd` | OK | actif |
| Premier SSH depuis admin | OK | après purge host key |
| Utilisateur admin dédié | Pas encore | doc suivante |
| Migration WAF / tunnel | Pas encore | docs suivantes |

---

## 7. Points à retenir (runbook)

1. **Rufus → mode DD** pour l’ISO Rocky en install offline.  
2. **Ne pas faire confiance** à la seule config réseau de l’installeur graphique → **nmcli** + autoconnect.  
3. **DNS public temporaire** si le DNS lab n’est pas encore utilisable.  
4. **Réinstall machine = nouvelles host keys** → `ssh-keygen -R <IP>` ; la clé utilisateur reste.  
5. Gateway = interface OPT du pare-feu ; pas de reprise aveugle des routes de l’ancien hôte sans revalider le besoin.

---

## 8. Suite

Document suivant : utilisateurs (`doo` limité / admin dédié), clés SSH dans `authorized_keys`, shell (Zsh / prompt), puis migration Docker / tunnel / WAF.

---

**Document fusionné – version portfolio / runbook – 15 août 2026**
