# 25 – Docker et migration cloudflared (G4)

**Projet :** Horus AIS – bastion  
**Date :** 6 août 2026 (fusion portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Objectif :** Installer Docker, puis migrer le tunnel Cloudflare (`cloudflared`) depuis l’ancien bastion vers le G4, **en utilisateur dédié non-root**, avant de poser le WAF.

> **Publication :** chemins de backup et identifiants de tunnel généralisés. En lab, remplacer par les noms / UUID réels.  
> Domaines publics (`horus-ais.com`, etc.) conservés tels quels.

---

## 1. Pourquoi cet ordre

| Étape | Rôle |
|-------|------|
| **1. Docker** | Prérequis pour BunkerWeb (conteneurs) |
| **2. cloudflared** | Tunnel sortant vers Cloudflare → sites joignables sans ouvrir 80/443 sur le FAI |
| **3. BunkerWeb** (doc 26) | Origine HTTP sur `localhost:80` que le tunnel doit atteindre |

Sans Docker, pas de WAF conteneurisé.  
Sans tunnel, le WAF resterait purement local.  
Sans WAF encore installé, le tunnel tourne déjà mais logue une erreur d’origine — **normale** (voir § 7).

On reprend aussi le principe de la fiche hardening tunnel : **processus cloudflared ≠ root**.

---

## 2. Installation de Docker

### 2.1 Préparation

```bash
sudo dnf -y update
sudo dnf -y install dnf-plugins-core
```

`dnf-plugins-core` fournit `config-manager` pour ajouter le dépôt officiel.

### 2.2 Dépôt et paquets

Sur Rocky 10, on utilise le dépôt Docker **centos** (compatible EL) :

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo dnf install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
```

### 2.3 Vérification

```bash
docker --version
# Exemple lab : Docker version 29.7.1

docker compose version
systemctl status docker --no-pager
# Active: active (running)
```

**Pourquoi `enable --now` :** démarre immédiatement **et** active au boot — indispensable pour un bastion qui doit remonter tunnel + WAF après reboot.

---

## 3. Installation de cloudflared

Paquet officiel Cloudflare (RPM) :

```bash
curl -L --output cloudflared.rpm \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-x86_64.rpm

sudo dnf install -y ./cloudflared.rpm
cloudflared --version
# Exemple lab : cloudflared version 2026.7.3
```

On installe depuis le `.rpm` local pour contrôler la version et éviter un dépôt tiers permanent si on ne le souhaite pas.

---

## 4. Utilisateur dédié non-root (hardening)

### Pourquoi pas root ?

Si `cloudflared` tourne en root et qu’une faille touche le binaire ou sa config, l’impact est maximal.  
Un utilisateur système **sans shell de login** limite la surface :

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin cloudflared

sudo mkdir -p /etc/cloudflared
sudo mkdir -p /var/log/cloudflared
sudo chown -R cloudflared:cloudflared /etc/cloudflared
sudo chown -R cloudflared:cloudflared /var/log/cloudflared
```

| Option `useradd` | Effet |
|------------------|--------|
| `--system` | UID système, pas un compte « humain » |
| `--no-create-home` | Pas de `/home/cloudflared` inutile |
| `--shell /usr/sbin/nologin` | Impossible de ouvrir une session interactive |

Vérification :

```bash
id cloudflared
# uid=…(cloudflared) gid=…(cloudflared) groups=…(cloudflared)
```

---

## 5. Récupération de la config depuis le backup

### 5.1 Fichiers nécessaires

Sur l’ancien bastion, le tunnel reposait sur :

- `/etc/cloudflared/config.yml` — hostnames + service d’origine
- `/etc/cloudflared/<credentials>.json` — secret du tunnel (à traiter comme un secret)

Sans le JSON credentials, un nouveau tunnel devrait être recréé côté Cloudflare Zero Trust.

### 5.2 Problème rencontré – montage du disque NTFS

**Symptôme :**

```text
mount: /mnt/backup: unknown filesystem type 'ntfs'.
```

**Cause :** Rocky Minimal n’inclut pas le support NTFS par défaut. Le backup lab était sur partition **NTFS** (disque externe préparé sous Windows).

**Correction :**

```bash
sudo dnf install -y epel-release
sudo dnf makecache
sudo dnf install -y ntfs-3g

sudo mkdir -p /mnt/backup
# Adapter le device après lsblk -f (exemple : /dev/sdb2)
sudo mount -t ntfs-3g /dev/sdX2 /mnt/backup
ls -la /mnt/backup
```

Toujours faire un `lsblk -f` **avant** de monter : le nom du device change selon l’ordre de branchement des disques.

Structure typique vue en lab :

```text
/mnt/backup/
└── <dossier-backup-ancien-bastion>/
    └── etc/cloudflared/
        ├── config.yml
        └── <tunnel-id>.json
```

### 5.3 Copie et droits

```bash
sudo cp /mnt/backup/<dossier-backup>/etc/cloudflared/config.yml /etc/cloudflared/
sudo cp /mnt/backup/<dossier-backup>/etc/cloudflared/<tunnel-id>.json /etc/cloudflared/

sudo chown -R cloudflared:cloudflared /etc/cloudflared
sudo chmod 600 /etc/cloudflared/*.json
sudo chmod 644 /etc/cloudflared/config.yml
```

**Pourquoi `600` sur le JSON :** contient les credentials du tunnel. Seul l’utilisateur `cloudflared` (et root) doit pouvoir le lire.

### 5.4 Forme du `config.yml` (schéma)

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: horus-ais.com
    service: http://localhost:80
  - hostname: www.horus-ais.com
    service: http://localhost:80
  - hostname: mrdoolux.horus-ais.com
    service: http://localhost:80
  - service: http_status:404
```

- Chaque hostname public est renvoyé vers **l’origine locale** `:80` (le WAF, une fois en place).
- La dernière règle `http_status:404` attrape le reste (bonne pratique Cloudflare Tunnel).

---

## 6. Service systemd (non-root)

```bash
sudo tee /etc/systemd/system/cloudflared.service > /dev/null << 'EOF'
[Unit]
Description=Cloudflare Tunnel
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=cloudflared
Group=cloudflared
ExecStart=/usr/bin/cloudflared --config /etc/cloudflared/config.yml tunnel run
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared --no-pager
```

| Directive | Intérêt |
|-----------|---------|
| `User=` / `Group=cloudflared` | Pas de root |
| `After=` / `Wants=network-online` | Attend le réseau avant de joindre Cloudflare |
| `Restart=on-failure` | Reprise automatique après coupure |
| `LimitNOFILE=65536` | Marge pour connexions concurrentes |

---

## 7. Problème rencontré – « Unable to reach the origin service »

**Symptôme dans les logs** (`journalctl -u cloudflared`) :

```text
ERR Unable to reach the origin service. The origin may be down...
localhost:80
```

**Cause :** le tunnel est **déjà connecté à Cloudflare**, mais rien n’écoute encore sur `localhost:80` (BunkerWeb n’est pas migré à ce stade).

**Ce n’est pas une panne du tunnel.** C’est l’état attendu entre la fin de la fiche 25 et la fiche 26.

| Élément | À ce stade |
|---------|------------|
| Processus `cloudflared` | Running |
| Session vers Cloudflare | OK |
| Origine `http://localhost:80` | Absente → erreurs périodiques normales |
| Sites publics | Répondront une fois le WAF up |

Après la doc 26, ces ERR disparaissent dès que BunkerWeb répond sur `:80`.

---

## 8. État final de cette étape

| Élément | Statut | Commentaire |
|---------|--------|-------------|
| Docker | OK | Prérequis WAF |
| cloudflared installé | OK | RPM officiel |
| User `cloudflared` non-root | OK | nologin |
| Config + credentials depuis backup | OK | droits 600 sur JSON |
| Service systemd enabled | OK | restart on-failure |
| Tunnel côté Cloudflare | OK | origin :80 en attente |

---

## 9. Points à retenir (runbook)

1. **Ordre :** Docker → cloudflared → WAF (pas l’inverse).  
2. **NTFS backup :** EPEL + `ntfs-3g` avant `mount` ; vérifier le device avec `lsblk -f`.  
3. **Credentials JSON = secret** → `chmod 600`, propriétaire `cloudflared`.  
4. **Service en `User=cloudflared`**, pas root.  
5. **ERR origin localhost:80** avant le WAF = normal, pas un rollback.  
6. Garder une copie offline des credentials tunnel (coffre / support chiffré) : sans eux, reconstruction plus lourde côté Cloudflare.

---

## 10. Suite

**Document 26 – Migration BunkerWeb multisite** : reprendre `/opt/bunkerweb` (compose + `.env` + `www/`), `docker compose up -d`, valider `localhost:80` puis les hostnames via le tunnel.

---

**Document fusionné – version portfolio / runbook – 15 août 2026**
