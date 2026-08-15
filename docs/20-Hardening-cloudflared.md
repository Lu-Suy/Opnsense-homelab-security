# 20 – Hardening cloudflared : utilisateur dédié non-root

**Date originale :** 31 juillet 2026  
**Machine :** Prodesk (Debian 12) – bastion  
**Version portfolio :** 15 août 2026 (sanitisée + enrichie)  
**Objectif :** Faire tourner `cloudflared` sous un utilisateur système non-root et renforcer les permissions des fichiers sensibles.

---

## Pourquoi un utilisateur dédié non-root ?

Par défaut, le service `cloudflared` tourne en **root**.

C’est fonctionnel, mais contraire au **principe du moindre privilège** :
- Si une vulnérabilité est exploitée dans `cloudflared`, l’attaquant obtient immédiatement les droits root.
- Les fichiers de credentials (très sensibles) sont accessibles au processus root sans restriction supplémentaire.

**Objectif du hardening :**
1. Créer un utilisateur système dédié (`cloudflared`)
2. Lui interdire toute connexion interactive (`nologin`)
3. Lui donner uniquement les droits nécessaires sur ses fichiers de configuration
4. Faire tourner le service sous cet utilisateur

Résultat : même en cas de compromission du processus, l’attaquant n’a plus les droits root.

---

## Étape 1 – Création de l’utilisateur dédié

### Commande tentée initialement
```bash
sudo useradd --system --no-create-home --shell /usr/bin/nologin cloudflared
```

**Résultat :**
```text
useradd: Warning: missing or non-executable shell '/usr/bin/nologin'
```

### Pourquoi ce warning ?
Sur Debian, le binaire `nologin` se trouve dans `/usr/sbin/nologin` et non dans `/usr/bin/nologin`.

### Vérifications effectuées
```bash
id cloudflared
which nologin
ls -l /usr/sbin/nologin /usr/bin/nologin 2>/dev/null
```

**Constat :**
- L’utilisateur a quand même été créé (`uid=999`)
- `nologin` existe bien dans `/usr/sbin/nologin`

### Correction du shell
```bash
sudo usermod -s /usr/sbin/nologin cloudflared
```

### Vérification finale
```bash
getent passwd cloudflared
```

**Sortie obtenue :**
```text
cloudflared:x:999:995::/home/cloudflared:/usr/sbin/nologin
```

**Explication de la sortie `getent passwd` :**

| Champ              | Valeur              | Signification                                      |
|--------------------|---------------------|----------------------------------------------------|
| Nom d’utilisateur  | `cloudflared`       | Nom du compte                                      |
| Mot de passe       | `x`                 | Le vrai hash est stocké dans `/etc/shadow`         |
| UID                | `999`               | Identifiant unique (compte système)                |
| GID                | `995`               | Groupe principal                                   |
| Commentaire (GECOS)| (vide)              | Champ libre, vide car compte technique             |
| Home               | `/home/cloudflared` | Chemin théorique (même s’il n’existe pas)          |
| Shell              | `/usr/sbin/nologin` | Connexion interactive interdite                    |

**Leçon retenue :**  
Toujours vérifier le chemin exact de `nologin` avec `ls /usr/sbin/nologin` avant de créer un utilisateur système sur Debian.

---

## Étape 2 – Permissions des fichiers sensibles

### 2.1 État initial
```bash
ls -la /etc/cloudflared/
```

**Ce qu’on cherche à voir :**
- Le fichier de configuration (`config.yml`)
- Le fichier de credentials (celui qui se termine par `.json`)
- Les propriétaires et permissions actuelles (normalement `root:root`)

![capture](../images/Pasted%20image%2020260731154412.png)

```text
-rw------- 1 root root 304 Jul 30 23:51 config.yml
-rw------- 1 root root 175 Jul 30 23:50 <TUNNEL-ID>.json
```

Les fichiers étaient déjà en `600` (seul root pouvait les lire), mais le service tournait encore en root.

### 2.2 Changement de propriétaire
```bash
sudo chown cloudflared:cloudflared /etc/cloudflared/config.yml
sudo chown cloudflared:cloudflared /etc/cloudflared/<TUNNEL-ID>.json
```

**Effet :**  
`chown utilisateur:groupe` change le propriétaire **et** le groupe des fichiers.

### 2.3 Vérification
```bash
ls -la /etc/cloudflared/
```

```text
-rw------- 1 cloudflared cloudflared 304 Jul 30 23:51 config.yml
-rw------- 1 cloudflared cloudflared 175 Jul 30 23:50 <TUNNEL-ID>.json
```

![capture](../images/Pasted%20image%2020260731154633.png)

---

## Étape 3 – Faire tourner le service sous l’utilisateur `cloudflared`

### 3.1 Service d’origine
```bash
systemctl cat cloudflared.service
```

![capture](../images/Pasted%20image%2020260731154922.png)

```ini
[Unit]
Description=Cloudflare Tunnel client
After=network-online.target
Wants=network-online.target

[Service]
TimeoutStartSec=0
Type=simple
ExecStart=/usr/bin/cloudflared --no-autoupdate --config /etc/cloudflared/config.yml tunnel run
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 3.2 Modification du service
```bash
sudo systemctl edit cloudflared.service --full
```

**Effet de la commande :**
- Ouvre l’éditeur (nano ou vim selon la config)
- Tu modifies le fichier de service complet

Ajouter dans la section `[Service]` :
```ini
User=cloudflared
Group=cloudflared
```

**Bloc final :**
```ini
[Service]
TimeoutStartSec=0
Type=simple
User=cloudflared
Group=cloudflared
ExecStart=/usr/bin/cloudflared --no-autoupdate --config /etc/cloudflared/config.yml tunnel run
Restart=always
RestartSec=5s
```

**Sauvegarde et quitte** l’éditeur :
- Si c’est **nano** : `Ctrl + O` puis `Entrée`, ensuite `Ctrl + X`
- Si c’est **vim** : `Échap`, puis `:wq` puis `Entrée`

### 3.3 Rechargement et redémarrage
```bash
sudo systemctl daemon-reload
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
```

![capture](../images/Pasted%20image%2020260731155432.png)

Le service est **actif** et le tunnel s’est correctement reconnecté.

### 3.4 Vérification que le processus tourne bien sous `cloudflared`
```bash
ps -o user,pid,cmd -p $(pgrep -f "cloudflared.*tunnel run")
```

![capture](../images/Pasted%20image%2020260731155829.png)

**Résultat :**
```text
USER     PID CMD
cloudfl+ 407939 /usr/bin/cloudflared --no-autoupdate --config /etc/cloudflared/config.yml tunnel run
```

`cloudfl+` est l’affichage tronqué de `cloudflared`.

**Commande alternative classique :**
```bash
ps aux | grep cloudflared | grep -v grep
```

---

## Explication des commandes de vérification

### `ps -o user,pid,cmd -p $(pgrep -f "cloudflared.*tunnel run")`

| Partie | Signification |
|--------|---------------|
| `pgrep -f "cloudflared.*tunnel run"` | Cherche le PID du processus dont la ligne de commande contient `cloudflared` et `tunnel run` |
| `$( ... )` | Substitution de commande : le PID trouvé est inséré |
| `ps -o user,pid,cmd -p ...` | Affiche uniquement les colonnes USER, PID et CMD pour ce PID |

### `ps aux | grep cloudflared | grep -v grep`

| Partie | Signification |
|--------|---------------|
| `ps aux` | Liste tous les processus |
| `\|` | Pipe (envoie la sortie vers la commande suivante) |
| `grep cloudflared` | Ne garde que les lignes contenant `cloudflared` |
| `grep -v grep` | Exclut la ligne du `grep` lui-même |

---

## Résultat final

| Point de contrôle                    | Statut |
|--------------------------------------|--------|
| Utilisateur système `cloudflared` créé | ✅ |
| Shell = `/usr/sbin/nologin`          | ✅ |
| Fichiers de config + credentials appartiennent à `cloudflared` | ✅ |
| Permissions restent en `600`         | ✅ |
| Service systemd utilise `User=` et `Group=` | ✅ |
| Processus tourne sous `cloudflared` (plus root) | ✅ |
| Tunnel reconnecté et fonctionnel     | ✅ |

**Hardening principal du tunnel Cloudflare réussi.**

---

**Document créé le 31 juillet 2026**  
**Document de référence – Hardening cloudflared (utilisateur non-root)**  
*Version portfolio sanitisée et enrichie – 15 août 2026*
