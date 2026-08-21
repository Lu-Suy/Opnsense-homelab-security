# 41 – Page vitrine Horus AIS + sous-domaine isa (contenu web)

**Projet :** Bastion Godmode / Horus AIS  
**Date :** 20–21 août 2026  
**Machine concernée :** G4 (Rocky Linux 10 – 10.0.10.11) – BunkerWeb 1.6.13  
**Références :** Fiches 21 (Multisite BunkerWeb), 26 (Migration BunkerWeb G4), 40 (Sites en 403 / Real-IP Cloudflare)  
**Statut :** Validé et en production

---

## 1. Contexte / Objectif

Jusqu’à cette fiche, `horus-ais.com` affichait une page de test minimale.  
Le lab était techniquement mature (pare-feu, WAF, SIEM, VPN, backups, monitoring), mais **la porte d’entrée publique manquait de contenu réel**.

**Objectifs de cette session :**

1. Déployer une **page vitrine professionnelle** sur `horus-ais.com`
2. Conserver l’ancien contenu sans le perdre
3. Créer le sous-domaine `isa.horus-ais.com` pour accueillir l’ancien index
4. Documenter proprement le déploiement (BunkerWeb Multisite + Cloudflare Tunnel)

La vitrine sert de **double usage** :
- Portfolio technique crédible (recruteurs / leads techniques)
- Identité de marque solide (base future High-Ticket Agency)

---

## 2. Nouvelle page vitrine

### 2.1 Contenu

| Élément | Description |
|---------|-------------|
| Image | Logo Horus AIS (fond sombre, œil d’Horus, typographie dorée) |
| Tagline | **Souveraineté. Contrôle. Confiance.** |
| Sous-titre | Infrastructure de défense personnelle. Lab cybersécurité documenté. |
| Boutons | **GitHub** → https://github.com/Lu-Suy/Opnsense-homelab-security<br>**LinkedIn** → https://www.linkedin.com/in/ludovic-suy-denis-a93569210/ |

### 2.2 Effet interactif (particulaires)

Effet vanilla JS (aucun framework) :

- **Survol** de l’image → souffle de particules (hiéroglyphes dorés + symboles Matrix)
- **Clic** → explosion / impulsion radiale de particules

Caractères utilisés :
- Hiéroglyphes égyptiens (𓂀 𓋹 𓆣 𓁹 …)
- Katakana + chiffres style Matrix (ア カ サ 0 1 …)
- Quelques symboles grecs (Σ Ω Δ Ψ)

Technique :
- Canvas en overlay (`z-index: 10`) au-dessus de l’image
- Particules générées au `mousemove` / `click`
- Fade-out + drift vers le haut / radial

Fichier source local : `horus-ais-vitrine.html` + `horus-ais.png`

### 2.3 Choix techniques

- Page **statique** (HTML + CSS + JS vanilla)
- Pas de React / TanStack / Grok Build pour la vitrine
- Objectif : simplicité, performance, maintenance facile derrière BunkerWeb
- L’application plus riche (console opérateur) reste prévue plus tard sur un autre sous-domaine

---

## 3. Migration de l’ancien index vers isa.horus-ais.com

### 3.1 Structure finale des sites

| URL | Dossier sur la G4 | Contenu |
|-----|-------------------|---------|
| `horus-ais.com` | `/opt/bunkerweb/www/horus-ais/` | Nouvelle vitrine |
| `isa.horus-ais.com` | `/opt/bunkerweb/www/isa/` | Ancien index (préservé) |
| `mrdoolux.horus-ais.com` | `/opt/bunkerweb/www/mrdoolux/` | Inchangé |

### 3.2 Actions réalisées

```bash
# Création du dossier et conservation de l'ancien index
sudo mkdir -p /opt/bunkerweb/www/isa
sudo cp /opt/bunkerweb/www/horus-ais/index.html /opt/bunkerweb/www/isa/index.html

# Placement de la nouvelle vitrine
sudo cp horus-ais-vitrine.html /opt/bunkerweb/www/horus-ais/index.html
sudo cp horus-ais.png          /opt/bunkerweb/www/horus-ais/horus-ais.png
```

L’ancien contenu n’a **jamais été écrasé** : il a d’abord été copié, puis seulement après la nouvelle page a été mise en place.

---

## 4. Modifications techniques

### 4.1 BunkerWeb – fichier `.env`

Ajouts / modifications dans `/opt/bunkerweb/.env` :

```env
SERVER_NAME=horus-ais.com mrdoolux.horus-ais.com isa.horus-ais.com

# Site principal
horus-ais.com_ROOT_FOLDER=/data/www/horus-ais
horus-ais.com_SERVE_FILES=yes

# Sous-domaine MrDoolux
mrdoolux.horus-ais.com_ROOT_FOLDER=/data/www/mrdoolux
mrdoolux.horus-ais.com_SERVE_FILES=yes

# Sous-domaine Isa
isa.horus-ais.com_ROOT_FOLDER=/data/www/isa
isa.horus-ais.com_SERVE_FILES=yes
```

### 4.2 cloudflared – ingress

Fichier `/etc/cloudflared/config.yml` :

```yaml
tunnel: fa29b757-5fb8-4867-ab4e-86970619be0d
credentials-file: /etc/cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json
ingress:
  - hostname: horus-ais.com
    service: http://localhost:80
  - hostname: www.horus-ais.com
    service: http://localhost:80
  - hostname: mrdoolux.horus-ais.com
    service: http://localhost:80
  - hostname: isa.horus-ais.com
    service: http://localhost:80
  - service: http_status:404
```

Redémarrage :

```bash
sudo systemctl restart cloudflared
```

### 4.3 DNS Cloudflare

| Type | Name | Target | Proxy |
|------|------|--------|-------|
| CNAME | `isa` | `fa29b757-5fb8-4867-ab4e-86970619be0d.cfargotunnel.com` | Proxied |

(Même cible que les autres enregistrements du tunnel.)

### 4.4 Rechargement BunkerWeb

Après chaque modification de `.env` :

```bash
cd /opt/bunkerweb
sudo docker compose down
sudo docker compose up -d
# Attendre la régénération du scheduler (~20-30 s)
sudo docker logs bw-scheduler --tail 20
```

Vérifier la présence de la ligne :
```
Generator successfully executed !
```

---

## 5. Point critique rencontré

### Symptôme

`https://isa.horus-ais.com` renvoyait **403 Forbidden**  
alors que `horus-ais.com` fonctionnait correctement.

### Diagnostic

Inspection de la configuration générée dans le conteneur :

```bash
sudo docker exec bunkerweb grep -i "ROOT_FOLDER" /etc/nginx/variables.env
```

Résultat observé :

```
horus-ais.com_ROOT_FOLDER=/data/www/horus-ais     ✅
mrdoolux.horus-ais.com_ROOT_FOLDER=/data/www/mrdoolux  ✅
isa.horus-ais.com_ROOT_FOLDER=/data/www               ❌
```

BunkerWeb servait le **dossier parent** `/data/www` au lieu de `/data/www/isa`.

### Cause

Absence (ou valeur incorrecte) de la variable spécifique :

```env
isa.horus-ais.com_ROOT_FOLDER=/data/www/isa
isa.horus-ais.com_SERVE_FILES=yes
```

### Correction

Ajout des deux lignes dans `.env`, puis `docker compose down && up -d`.  
Après régénération, la variable était correcte :

```
isa.horus-ais.com_ROOT_FOLDER=/data/www/isa
isa.horus-ais.com_SERVE_FILES=yes
```

Le site a alors répondu normalement.

### Leçon

En Multisite BunkerWeb, **chaque hostname** doit avoir explicitement :
- `HOSTNAME_ROOT_FOLDER=...`
- `HOSTNAME_SERVE_FILES=yes`

Le simple ajout dans `SERVER_NAME` ne suffit pas toujours à dériver correctement le root.

---

## 6. Validation

| Test | Résultat |
|------|----------|
| https://horus-ais.com | Vitrine affichée, image + tagline + boutons OK |
| Effet hover (particules) | OK |
| Effet clic (explosion) | OK |
| https://isa.horus-ais.com | Ancien index affiché |
| https://mrdoolux.horus-ais.com | Inchangé / OK |
| Logs BunkerWeb | Plus de 403 systématique sur isa |
| cloudflared | Ingress `isa.horus-ais.com` actif |

---

## 7. État final

```
horus-ais.com          →  Nouvelle vitrine (porte d’entrée)
isa.horus-ais.com      →  Ancien index (conservé)
mrdoolux.horus-ais.com →  Inchangé
```

**Socle public :**
- Identité visuelle en place
- Contenu réel déployé
- Ancien contenu préservé
- Configuration Multisite + Tunnel documentée

### Prochaines pistes (hors scope de cette fiche)

- Enrichissement éventuel de la vitrine (section courte « lab », lien documentation)
- Déploiement futur d’une console opérateur sur un autre sous-domaine
- Alertes e-mail (Fail2ban / Wazuh / OPNsense)
- Contenu métier plus développé selon orientation recruteur / clients

---

**Fiche clôturée le 21 août 2026.**  
Vitrine en production. Ancien index préservé sur `isa.horus-ais.com`.
