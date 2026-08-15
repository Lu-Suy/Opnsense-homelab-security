# 24 – Users, SSH et shell (G4)

**Projet :** Horus AIS – bastion  
**Date :** 5–6 août 2026 (fusion portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Objectif :** Mettre en place une séparation claire des comptes, l’auth SSH par clé, et un shell de travail stable **avant** la migration des services (Docker, tunnel, WAF).

> **Publication :** IP de management masquée en `10.0.10.y`. En lab, utiliser l’IP réelle du plan d’adressage.  
> Clé utilisateur nommée `id_ed25519` (sans suffixe lab).

---

## 1. Pourquoi cette étape avant les services

Sans comptes et accès SSH propres :

- on administre tout en `root` ou avec un seul compte trop privilégié ;
- on ne peut pas appliquer plus tard un durcissement SSH (`AllowUsers`, clés only) sans se bloquer dehors ;
- on mélange usage quotidien et actions d’administration.

On fixe d’abord **qui** peut se connecter et **avec quels droits**, ensuite seulement on pose Docker / tunnel / WAF.

---

## 2. Architecture des utilisateurs

| Utilisateur | Rôle | Sudo | Shell |
|-------------|------|------|-------|
| **`doo`** | Compte limité (usage courant, pas d’admin) | Non | Zsh |
| **`hyper_doo`** | Administrateur du bastion | Oui (`wheel`) | Zsh |
| **`root`** | Urgence / séries de commandes admin | — | Zsh |

### Pourquoi deux comptes non-root ?

- **Moindre privilège :** un compte sans sudo limite l’impact d’une session compromise ou d’une erreur de manip.
- **`hyper_doo` + `wheel` :** sur Rocky/RHEL, le groupe **`wheel`** est l’équivalent du groupe `sudo` sous Debian/Ubuntu. C’est lui qui autorise `sudo`.
- **`root` en direct :** réservé aux phases où enchaîner `sudo` sur chaque commande devient contre-productif (`sudo -i`), pas au quotidien.

---

## 3. Création de `hyper_doo`

```bash
useradd -m -s /bin/bash hyper_doo
passwd hyper_doo
usermod -aG wheel hyper_doo
```

| Option | Effet |
|--------|--------|
| `-m` | Crée le home `/home/hyper_doo` |
| `-s /bin/bash` | Shell initial (passé à Zsh ensuite) |
| `wheel` | Droit d’utiliser `sudo` |

**Vérification :**

```bash
id hyper_doo
groups hyper_doo
```

Attendu : présence de `wheel` (souvent gid 10).

---

## 4. Séparation des privilèges – retirer `doo` de `wheel`

### Problème / contexte

Pendant l’installation graphique Rocky, le premier utilisateur (`doo`) est souvent créé **déjà administrateur** (membre de `wheel`).  
Si on laisse ainsi, l’architecture « doo limité / hyper_doo admin » n’existe que sur le papier.

### Correction

```bash
gpasswd -d doo wheel
id doo
groups doo
```

Attendu : `doo` **sans** `wheel`.

On **garde** ensuite une clé SSH pour `doo` : se connecter en compte limité reste utile (lecture, tests, moindre risque). On **retire** seulement le sudo.

---

## 5. Authentification SSH par clé

### Principe

- La **clé privée** reste sur le poste d’administration (Windows).
- Seule la **clé publique** est copiée dans `~/.ssh/authorized_keys` sur le bastion.
- Même clé publique pour `doo` et `hyper_doo` (un poste admin, deux comptes serveur) — acceptable en lab mono-opérateur ; en équipe on séparerait les clés.

### Droits obligatoires (sinon sshd ignore la clé)

| Chemin | Mode | Pourquoi |
|--------|------|----------|
| `~/.ssh/` | `700` | Dossier accessible uniquement au propriétaire |
| `~/.ssh/authorized_keys` | `600` | Fichier non lisible par les autres |

Sans ces droits, OpenSSH refuse l’auth par clé (message du type *Authentication refused: bad ownership or modes*).

### 5.1 Compte `hyper_doo`

```bash
mkdir -p /home/hyper_doo/.ssh
chmod 700 /home/hyper_doo/.ssh

# Coller la clé publique du poste admin (une seule ligne ssh-ed25519 ...)
echo "ssh-ed25519 AAAA... commentaire-poste-admin" >> /home/hyper_doo/.ssh/authorized_keys

chmod 600 /home/hyper_doo/.ssh/authorized_keys
chown -R hyper_doo:hyper_doo /home/hyper_doo/.ssh
```

### 5.2 Compte `doo`

```bash
mkdir -p /home/doo/.ssh
chmod 700 /home/doo/.ssh

echo "ssh-ed25519 AAAA... commentaire-poste-admin" | tee /home/doo/.ssh/authorized_keys

chmod 600 /home/doo/.ssh/authorized_keys
chown -R doo:doo /home/doo/.ssh
```

### 5.3 Test depuis le poste admin

```powershell
ssh -i $HOME\.ssh\id_ed25519 hyper_doo@10.0.10.y
ssh -i $HOME\.ssh\id_ed25519 doo@10.0.10.y
```

Attendu : plus de demande de **mot de passe compte** (la passphrase de la clé privée, elle, peut encore être demandée — c’est normal et souhaitable).

> En publication, la clé publique complète n’est pas reproduite ici. En lab, utiliser la ligne issue de `id_ed25519.pub`.

---

## 6. Environnement shell – Zsh + Starship

### 6.1 Pourquoi Zsh + Starship

- Autocomplétion et confort sur un bastion où l’on passe du temps en CLI.
- Prompt minimal mais **lisible** (heure + indicateur de succès/échec) pour les captures et la doc.
- Distinction visuelle **admin vs root** (symbole différent) : moins d’erreurs « je croyais être en user ».

### 6.2 Installation

```bash
dnf install -y zsh tar gzip curl git

chsh -s /bin/zsh hyper_doo
chsh -s /bin/zsh doo
chsh -s /bin/zsh root
```

**Point d’attention – `chsh` :** indiquer le **chemin complet** (`/bin/zsh`), pas `$(which zsh)` si `which` n’est pas installé (Minimal Install).

```bash
# Starship (binaire dans /usr/local/bin en général)
curl -sS https://starship.rs/install.sh | sh
```

**Point d’attention – dépendances :** l’installateur Starship a besoin de `tar` (et souvent `gzip`). Sur Minimal Install, les installer **avant** le script.

### 6.3 Plugins utiles (exemple `hyper_doo`)

```bash
mkdir -p /home/hyper_doo/.zsh
git clone https://github.com/zsh-users/zsh-autosuggestions \
  /home/hyper_doo/.zsh/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  /home/hyper_doo/.zsh/zsh-syntax-highlighting
chown -R hyper_doo:hyper_doo /home/hyper_doo/.zsh
```

Brancher ces plugins dans `~/.zshrc` (source des scripts + `eval "$(starship init zsh)"`).

### 6.4 Prompt Starship (minimal)

**`hyper_doo`** – indicateur type éclair :

```toml
format = "$time $character"

[time]
disabled = false
format = "[$time](bold bright-black)"
time_format = "%H:%M"

[character]
success_symbol = "[⚡](bold yellow)"
error_symbol = "[⚡](bold red)"
```

**`root`** – indicateur distinct (tête de mort) :

```toml
format = "$time $character"

[time]
disabled = false
format = "[$time](bold bright-black)"
time_format = "%H:%M"

[character]
success_symbol = "[☠](bold white)"
error_symbol = "[☠](bold red)"
```

Fichiers typiques : `~/.config/starship.toml` pour chaque compte.

### 6.5 Problèmes rencontrés (shell / terminal)

| Symptôme | Cause | Correction |
|----------|--------|------------|
| `tar: command not found` pendant install Starship | Minimal Install sans `tar` | `dnf install -y tar gzip` puis relancer |
| `chsh: shell must be a full path name` | Chemin incomplet / `which` absent | `chsh -s /bin/zsh <user>` |
| Symboles (⚡ ☠) illisibles sous Windows | Police terminal sans glyphs | Windows Terminal + police type **Nerd Font** (Meslo, Fira Code Nerd, etc.) |
| Après `sudo -i`, Starship absent | `PATH` root sans `/usr/local/bin` | Ajouter `/usr/local/bin` au PATH de root ou symlink ; vérifier `~/.zshrc` root |

---

## 7. Sudo sans mot de passe pour `hyper_doo`

```bash
echo "hyper_doo ALL=(ALL) NOPASSWD: ALL" | tee /etc/sudoers.d/hyper_doo
chmod 440 /etc/sudoers.d/hyper_doo
```

### Pourquoi ce choix (lab)

- Évite de retaper le mot de passe à chaque `sudo` pendant les migrations longues.
- Fichier dédié dans `sudoers.d/` (pas d’édition directe de `/etc/sudoers`) → plus sûr aux mises à jour.

### Limite (esprit prod / entretien)

`NOPASSWD: ALL` est **large**. En production on restreindrait les commandes autorisées.  
Ici le bastion est mono-opérateur, derrière pare-feu, auth SSH par clé : acceptable en lab, à **revoir** avant exposition plus large.

### Comportement retenu

| Situation | Pratique |
|-----------|----------|
| Quotidien | Rester en `hyper_doo`, `sudo` au besoin |
| Longue série admin | `sudo -i` (shell root, prompt distinct) |
| À éviter | Wrappers qui exécutent tout en root en silence |

---

## 8. État final de cette étape

| Élément | Statut | Commentaire |
|---------|--------|-------------|
| `hyper_doo` + `wheel` | OK | Admin dédié |
| `doo` hors `wheel` | OK | Compte limité |
| Clés SSH `doo` + `hyper_doo` | OK | Depuis poste admin |
| Zsh + Starship | OK | Prompts distincts user / root |
| Sudo NOPASSWD `hyper_doo` | OK | Lab ; à durcir plus tard si besoin |
| Auth mot de passe SSH encore possible | Oui | Coupée au durcissement SSH (doc 28) |

---

## 9. Points à retenir (runbook)

1. Sur Rocky, **admin = groupe `wheel`**, pas le groupe `sudo` Debian.  
2. Le premier user de l’install graphique est souvent **déjà** admin → le **retirer** de `wheel` si l’archi le demande.  
3. Droits `.ssh` **700** / `authorized_keys` **600** + bon `chown`, sinon clé ignorée.  
4. Minimal Install : installer **`tar`/`gzip`** avant Starship ; **`chsh -s /bin/zsh`**.  
5. `NOPASSWD` = confort lab, pas un modèle prod par défaut.  
6. Prompt différent root vs user = filet anti-erreur humain.

---

## 10. Suite

Document suivant : **Docker** puis migration **cloudflared** (tunnel) et **BunkerWeb** (WAF), toujours sous le compte admin et avec le réseau management déjà stable (doc 23).

---

**Document fusionné – version portfolio / runbook – 15 août 2026**
