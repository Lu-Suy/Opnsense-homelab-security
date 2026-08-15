# 28 – Durcissement SSH (G4)

**Projet :** Horus AIS – bastion  
**Date :** 10 août 2026 (sanitisé portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Objectif :** Durcir le serveur SSH selon les bonnes pratiques (esprit ANSSI) : désactiver l’authentification par mot de passe, restreindre root, limiter les fonctionnalités inutiles et les tentatives de connexion.

> **Publication :** IP de management masquée en `10.0.10.y`. Clé nommée `id_ed25519`.

---

## 1. Contexte

Après la migration du bastion sur le G4 (docs 23 à 27), le service SSH fonctionnait correctement mais avec une configuration trop permissive par défaut :

| Paramètre | Valeur avant | Risque |
|-----------|--------------|--------|
| `PasswordAuthentication` | `yes` | Brute-force possible |
| `PermitRootLogin` | `yes` | Attaque directe sur root |
| `X11Forwarding` | `yes` | Surface d’attaque inutile |
| `AllowTcpForwarding` | `yes` | Tunneling non contrôlé |
| `MaxAuthTries` | `6` | Trop permissif |

L’objectif était d’appliquer un durcissement **réel**, dans l’esprit des recommandations ANSSI, tout en gardant un accès administrateur fiable **par clé SSH**.

---

## 2. Principes de durcissement appliqués

| Règle (esprit ANSSI / bonne pratique) | Paramètre | Valeur retenue |
|---------------------------------------|-----------|----------------|
| Pas d’auth par mot de passe | `PasswordAuthentication` | `no` |
| Root uniquement par clé | `PermitRootLogin` | `prohibit-password` |
| Comptes autorisés seulement | `AllowUsers` | `doo hyper_doo` |
| Limitation des tentatives | `MaxAuthTries` | `3` |
| Sessions inactives | `ClientAliveInterval` / `CountMax` | `300` / `2` |
| Pas de X11 | `X11Forwarding` | `no` |
| Pas de forwarding TCP | `AllowTcpForwarding` | `no` |
| Logs détaillés | `LogLevel` | `VERBOSE` |

---

## 3. Sauvegarde préalable

```bash
# Sauvegarde de la configuration serveur SSH
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%F)
```

**Explication :**  
Toujours garder une copie avant modification.  
Attention : `/etc/ssh/ssh_config` = configuration **client** ; `/etc/ssh/sshd_config` = configuration **serveur**.

---

## 4. État initial (avant durcissement)

On ne se fie pas au seul fichier texte : on lit la configuration **effective** après parsing.

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|maxauthtries|x11forwarding|allowtcpforwarding|loglevel'
```

**Pourquoi `sshd -T` ?**  
Il affiche les valeurs réellement appliquées (fichiers principaux + `sshd_config.d/`), pas seulement ce qui est écrit dans un seul fichier.

Capture lab – état **avant** durcissement :

![sshd -T – configuration effective avant durcissement](../images/Pasted%20image%2020260810182759.png)

Résultat typique constaté :

```text
maxauthtries 6
permitrootlogin yes
passwordauthentication yes
x11forwarding yes
allowtcpforwarding yes
loglevel INFO
```

C’est le point de départ : trop ouvert pour un bastion exposé même derrière le pare-feu lab.

---

## 5. Création du fichier de durcissement

Sur les systèmes modernes (RHEL/Rocky 9+), la bonne pratique est d’utiliser un fichier dans `/etc/ssh/sshd_config.d/` plutôt que de modifier massivement le fichier principal.

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null << 'EOF'
# ============================================
# Durcissement SSH - Bastion Horus AIS (G4)
# Aligné sur les bonnes pratiques type ANSSI
# ============================================

# Authentification
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no

# Root
PermitRootLogin prohibit-password

# Utilisateurs autorisés uniquement
AllowUsers doo hyper_doo

# Limitation des tentatives
MaxAuthTries 3
LoginGraceTime 30

# Sessions
ClientAliveInterval 300
ClientAliveCountMax 2
MaxSessions 5

# Fonctionnalités inutiles
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no
GatewayPorts no

# Logs
LogLevel VERBOSE

# Protocole
Protocol 2
EOF
```

**Explication des directives importantes :**

- `PasswordAuthentication no` : plus aucune connexion par mot de passe
- `PermitRootLogin prohibit-password` : root uniquement avec une clé SSH
- `AllowUsers doo hyper_doo` : seuls ces deux comptes sont acceptés
- `MaxAuthTries 3` : 3 tentatives maximum
- Forwarding désactivé : réduction de la surface d’attaque
- `LogLevel VERBOSE` : meilleurs logs pour l’audit / Fail2ban

---

## 6. Problème rencontré : ordre de lecture des fichiers

### Symptôme

Après `systemctl reload sshd`, certains paramètres n’étaient **pas** appliqués :

```text
permitrootlogin yes          ← toujours yes
x11forwarding yes            ← toujours yes
passwordauthentication no    ← OK
```

### Diagnostic

```bash
grep -rniE 'permitrootlogin|x11forwarding' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Résultat :

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf:3:PermitRootLogin yes
/etc/ssh/sshd_config.d/50-redhat.conf:10:X11Forwarding yes
/etc/ssh/sshd_config.d/99-hardening.conf:14:PermitRootLogin prohibit-password
/etc/ssh/sshd_config.d/99-hardening.conf:29:X11Forwarding no
```

### Cause racine

OpenSSH applique la **première** occurrence d’une directive.  
Les fichiers dans `sshd_config.d/` sont lus **par ordre alphabétique** :

1. `01-permitrootlogin.conf` → `PermitRootLogin yes` (**gagne**)
2. `50-redhat.conf` → `X11Forwarding yes` (**gagne**)
3. `99-hardening.conf` → nos valeurs (**trop tard, ignorées**)

C’est un comportement classique sur Rocky/RHEL 9 et 10.

---

## 7. Pourquoi on n’a pas modifié directement les fichiers système

On aurait pu éditer :

- `/etc/ssh/sshd_config.d/01-permitrootlogin.conf`
- `/etc/ssh/sshd_config.d/50-redhat.conf`

**Pourquoi on ne l’a pas fait :**

| Approche | Avantage | Inconvénient |
|----------|----------|--------------|
| Modifier les fichiers système (`01-`, `50-`) | Direct | Risque d’être **écrasé** lors d’une mise à jour du paquet `openssh` |
| Créer un fichier prioritaire `00-hardening.conf` | Persistant, propre, clair | Aucun (meilleure pratique) |

**Choix retenu :**  
Renommer notre fichier en `00-hardening.conf` pour qu’il soit lu **en premier**.

```bash
sudo mv /etc/ssh/sshd_config.d/99-hardening.conf /etc/ssh/sshd_config.d/00-hardening.conf
```

Ainsi :

- Nos directives passent avant celles de Red Hat / Rocky
- Une mise à jour système n’écrase pas notre durcissement
- La configuration reste lisible et documentée dans un seul fichier dédié

C’est la méthode recommandée : **override par priorité de nom**, sans toucher aux fichiers fournis par le paquet.

---

## 8. Application et validation

```bash
# Vérifier la syntaxe (obligatoire avant reload)
sudo sshd -t

# Recharger sans couper les sessions existantes
sudo systemctl reload sshd

# Vérifier les valeurs effectives
sudo sshd -T | grep -E 'permitrootlogin|x11forwarding|passwordauthentication|maxauthtries|allowusers|loglevel'
```

**`sshd -t` :** si la syntaxe est invalide, la commande signale l’erreur et on **ne** recharge **pas**.  
**`reload` :** applique la nouvelle config sans déconnecter les sessions déjà ouvertes (contrairement à `restart`).

### Résultat final obtenu

```text
permitrootlogin without-password
passwordauthentication no
x11forwarding no
maxauthtries 3
loglevel VERBOSE
allowusers doo hyper_doo
```

Note : `without-password` est l’ancien synonyme de `prohibit-password`.  
Les deux signifient : **root autorisé uniquement par clé SSH**.

---

## 9. Tests de validation

Depuis le poste d’administration :

```powershell
# Doit fonctionner (clé SSH)
ssh -i $HOME\.ssh\id_ed25519 hyper_doo@10.0.10.y
ssh -i $HOME\.ssh\id_ed25519 doo@10.0.10.y
```

Comportement attendu :

- Demande de la passphrase de la clé privée (si elle en a une)
- Connexion réussie
- Tentative de connexion par mot de passe → **refusée**

---

## 10. État final – 10 août 2026

| Élément | Statut |
|---------|--------|
| Auth par mot de passe | Désactivée |
| Root par clé uniquement | OK |
| AllowUsers (doo + hyper_doo) | OK |
| X11Forwarding | Désactivé |
| AllowTcpForwarding | Désactivé |
| MaxAuthTries = 3 | OK |
| LogLevel VERBOSE | OK |
| Fichier prioritaire `00-hardening.conf` | OK |
| Connexions par clé validées | OK |

---

## 11. Points d’attention

1. **Initramfs / dracut-sshd**  
   Le déverrouillage distant LUKS (doc 27) utilise sa propre configuration SSH dans l’initramfs.  
   Le durcissement de cette fiche concerne le **sshd du système démarré**.

2. **Recovery**  
   En cas de problème, la sauvegarde est disponible :
   ```bash
   /etc/ssh/sshd_config.bak-YYYY-MM-DD
   ```

3. **Fail2ban**  
   Prochaine étape logique : installer Fail2ban pour bannir automatiquement les IP qui multiplient les échecs d’authentification.

---

## 12. Commandes de contrôle rapide

```bash
# Config effective
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|allowusers|x11forwarding|maxauthtries'

# Sessions SSH en cours
ss -tnp | grep :22

# Logs SSH récents
journalctl -u sshd -n 30 --no-pager
```

---

**Document validé en lab – version portfolio / runbook – 15 août 2026**
