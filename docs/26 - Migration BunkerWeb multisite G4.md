# 26 – Migration BunkerWeb multisite (G4)

**Projet :** Horus AIS – bastion  
**Date :** 6 août 2026 (réalisé) – fusion portfolio 15 août 2026  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Objectif :** Reposer BunkerWeb en mode multisite sur le G4, derrière le tunnel Cloudflare déjà migré (doc 25), à partir d’une **copie maîtrisée** de l’ancien bastion — pas d’une improvisation.

> **Publication :** chemins de backup généralisés. Domaines publics conservés.

---

## 1. Contexte

À l’issue de la doc **25** :

- Docker tourne sur le G4 ;
- `cloudflared` tourne en user non-root et pointe les hostnames vers `http://localhost:80` ;
- les logs tunnel signalent encore *Unable to reach the origin* → **attendu** tant que rien n’écoute sur `:80`.

Il manque l’**origine** : BunkerWeb (WAF + fichiers statiques), précédemment en production sur l’ancien mini-PC (Debian).

**Sites concernés :**

- `horus-ais.com`
- `www.horus-ais.com`
- `mrdoolux.horus-ais.com`

---

## 2. D’où vient le backup ? (point souvent oublié)

### 2.1 Ce qui a été fait (et où c’est écrit)

Il n’existe **pas** de fiche numérotée dédiée uniquement au backup de l’ancien bastion.  
La bascule a été préparée ainsi :

| Moment | Action | Référence |
|--------|--------|-----------|
| Avant install Rocky sur le G4 | Copie des éléments critiques de l’ancien hôte vers un **disque externe** | Évoqué en **doc 23** (§ backup avant migration) |
| Pendant la doc 25 | Montage de ce disque sur le G4 (`ntfs-3g`), usage pour `cloudflared` | **Doc 25** (§ récupération config) |
| Doc 26 | Même montage / même arborescence pour `/opt/bunkerweb` | **Cette fiche** |

En pratique, le backup lab était un arbre type « copie de l’ancien système » (ou extrait structuré) sur partition **NTFS**, dossier daté du jour de bascule, par exemple :

```text
/mnt/backup/<dossier-backup-ancien-bastion>/
├── etc/cloudflared/     ← déjà utilisé en doc 25
└── opt/bunkerweb/       ← objet de la doc 26
    ├── docker-compose.yml
    ├── .env
    ├── www/
    └── (éventuels .bak)
```

### 2.2 Contenu minimal indispensable pour BunkerWeb

Sans ces éléments, on ne « migre » pas : on réinstalle à blanc et on reperd la config multisite validée.

| Élément | Rôle |
|---------|------|
| `docker-compose.yml` | Services, images, ports, volumes, réseaux |
| `.env` | Multisite, `SERVER_NAME`, chemins par hostname, options sécu |
| `www/` | Fichiers servis (`horus-ais/`, `mrdoolux/`, …) |

### 2.3 Principe retenu

1. **Ne pas casser** l’ancien bastion tant que le G4 n’a pas prouvé le service.  
2. **Copier** la config connue bonne, plutôt que la réécrire de mémoire.  
3. **Vérifier** la structure sur le support de backup **avant** le `docker compose up`.

> Pour un prochain lab : une fiche « Backup à froid / inventaire fichiers critiques » mériterait d’exister **avant** la migration OS. Ici on documente a posteriori ce qui a réellement servi.

---

## 3. Prérequis sur le G4

- Docker + plugin Compose opérationnels (doc 25)
- Tunnel `cloudflared` actif (même doc)
- Disque de backup **monté** (voir doc 25 si besoin : EPEL, `ntfs-3g`, `lsblk -f`, `mount -t ntfs-3g … /mnt/backup`)

---

## 4. Vérification du backup **avant** copie

On ne lance pas Compose « en aveugle ». Checklist lab :

```bash
# Adapter le chemin du dossier backup
BACKUP=/mnt/backup/<dossier-backup-ancien-bastion>

ls -la "$BACKUP/opt/bunkerweb/"
ls -la "$BACKUP/opt/bunkerweb/www/"
find "$BACKUP/opt/bunkerweb/www/" -type f

# Lire (sans modifier) la config
less "$BACKUP/opt/bunkerweb/docker-compose.yml"
less "$BACKUP/opt/bunkerweb/.env"
```

### Structure attendue sous `www/`

```text
www/
├── horus-ais/
│   └── index.html   # (et assets éventuels)
└── mrdoolux/
    └── index.html
```

### Points de contrôle dans le `.env` (schéma)

```bash
MULTISITE=yes
SERVER_NAME=horus-ais.com mrdoolux.horus-ais.com
DISABLE_DEFAULT_SERVER=yes
SERVE_FILES=yes
ROOT_FOLDER=/data/www

horus-ais.com_ROOT_FOLDER=/data/www/horus-ais
horus-ais.com_SERVE_FILES=yes

mrdoolux.horus-ais.com_ROOT_FOLDER=/data/www/mrdoolux
mrdoolux.horus-ais.com_SERVE_FILES=yes
```

Si `MULTISITE` / `SERVER_NAME` / chemins `*_ROOT_FOLDER` sont absents ou incohérents avec `www/`, corriger **sur une copie de travail**, pas en prod improvisée.

### `docker-compose.yml` (résumé lab)

| Élément | Valeur typique retenue |
|---------|-------------------------|
| Image | `bunkerity/bunkerweb:1.6.10` (+ scheduler associé) |
| Ports hôte | `80:8080`, `443:8443` (TCP/UDP selon compose) |
| Volume sites | `./www:/data/www:ro` |
| Services | `bunkerweb`, `bw-scheduler`, souvent `docker-socket-proxy` |

Le tunnel Cloudflare n’a besoin que de **`:80`** en origine HTTP pour les sites actuels.

---

## 5. Migration réalisée

```bash
sudo mkdir -p /opt/bunkerweb

# Copie récursive en préservant attributs
sudo cp -a /mnt/backup/<dossier-backup-ancien-bastion>/opt/bunkerweb/. /opt/bunkerweb/
sudo chown -R root:root /opt/bunkerweb

# Contrôle post-copie
ls -la /opt/bunkerweb/
ls -la /opt/bunkerweb/www/
test -f /opt/bunkerweb/docker-compose.yml && test -f /opt/bunkerweb/.env && echo "compose+env OK"

cd /opt/bunkerweb
sudo docker compose up -d
sudo docker ps
```

**Pourquoi `cp -a` :** préserve permissions et structure ; évite de casser un volume ou un script déjà mode correct.

### Résultat conteneurs (lab)

| Conteneur | Statut observé |
|-----------|----------------|
| **bunkerweb** | Up (healthy) |
| **bw-scheduler** | Up |
| **docker-socket-proxy** | Up |

Capture lab (`docker ps` juste après le `compose up`) :

![docker ps – BunkerWeb, scheduler et proxy up](../images/Pasted%20image%2020260805080711.png)

---

## 6. Vérification de l’aboutissement (assurance)

L’objectif n’est pas seulement « des conteneurs verts », c’est **preuve** que l’origine répond et que le tunnel sert les bons vhosts.

### 6.1 Local (sur le G4)

```bash
curl -I http://127.0.0.1
curl -I -H "Host: horus-ais.com" http://127.0.0.1
curl -I -H "Host: mrdoolux.horus-ais.com" http://127.0.0.1
```

Attendu : réponses HTTP 200 (ou 3xx maîtrisés), **pas** connexion refused.  
Le header `Host` simule le vhost multisite comme le fera le tunnel.

### 6.2 Via Cloudflare (extérieur)

Navigateur ou `curl` :

- `https://horus-ais.com`
- `https://mrdoolux.horus-ais.com`

Attendu : pages servies (contenu de `www/…`).

### 6.3 Côté tunnel

Les ERR *Unable to reach the origin service* / `localhost:80` doivent **cesser** ou fortement diminuer une fois BunkerWeb healthy sur `:80`.

```bash
sudo systemctl status cloudflared --no-pager
sudo journalctl -u cloudflared -n 30 --no-pager
```

### 6.4 Checklist de clôture

| Contrôle | OK ? |
|----------|------|
| Backup monté et structure `opt/bunkerweb` relue avant copie | |
| `docker-compose.yml` + `.env` + `www/` présents sous `/opt/bunkerweb` | |
| `docker ps` : bunkerweb healthy | |
| `curl` local avec `Host:` des deux sites | |
| HTTPS public via tunnel | |
| Plus d’erreur origin bloquante dans les logs cloudflared | |

Cette checklist est la **preuve d’aboutissement** : on ne déclare pas la migration terminée sur la seule foi d’un `compose up`.

---

## 7. État final

| Élément | Statut | Commentaire |
|---------|--------|-------------|
| Docker | OK | Doc 25 |
| cloudflared non-root | OK | Doc 25 |
| BunkerWeb multisite | OK | Config issue du backup ancien bastion |
| Pages statiques | OK | Copiées sous `/opt/bunkerweb/www` |
| Sites publics | OK | Tunnel → localhost:80 → BunkerWeb |

### Volontairement reporté (à ce stade)

- Durcissement SSH / Fail2ban (docs 28–29)
- Mode Block / ModSecurity plus strict sur BunkerWeb
- Wazuh, WireGuard
- Contenu « production » des pages (au-delà des index de lab)

---

## 8. Points à retenir (runbook)

1. **Pas de migration WAF sans inventaire backup** : compose + env + www.  
2. Le backup Prodesk / ancien bastion a été fait **avant** Rocky sur le G4 ; montage NTFS documenté en doc 25.  
3. **Vérifier sur le support** avant `cp`, **vérifier en local puis public** après `up`.  
4. Tunnel OK + origin down = normal en doc 25 ; tunnel OK + origin up = succès doc 26.  
5. Une fiche « backup critique » standalone reste une amélioration doc utile pour la prochaine bascule.

---

## 9. Suite

**Document 27** – déverrouillage distant LUKS (SSH en initramfs), pour ne plus dépendre d’un écran local au boot du bastion chiffré.

---

**Document fusionné – version portfolio / runbook – 15 août 2026**
