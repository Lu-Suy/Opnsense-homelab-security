# 08 – Installation BunkerWeb (Version Finale Fonctionnelle)

**Date originale** : 17 mai 2026  
**Statut** : ✅ **Fonctionnel** – Page statique personnalisée affichée  
**Version portfolio** : 15 août 2026 (sanitization)  
**Objectif atteint** : BunkerWeb sert correctement la page `index.html` sur l’IP services du bastion

> **Note portfolio** :  
> Snapshot de la configuration qui a fonctionné en mai 2026 (ancien bastion).  
> Les IPs et noms ont été généralisés. Contenu technique et explications conservés intégralement.

---

### Configuration finale qui marche (la bonne)

#### 1. Fichier `.env`

```env
# ==================== Variables BunkerWeb ====================
MULTISITE=yes
SERVER_NAME=10.0.10.x bastion
DEFAULT_SERVER=10.0.10.x
DISABLE_DEFAULT_SERVER=no

# === Réseau & DNS ===
DNS_RESOLVERS=10.0.0.1

# === API interne ===
API_WHITELIST_IP=127.0.0.1 172.16.0.0/12 10.0.0.0/8

# === FICHIERS STATIQUES (clé du succès) ===
SERVE_FILES=yes
ROOT_FOLDER=/data/www

# === Sécurité en mode test ===
SECURITY_MODE=detect
LOG_LEVEL=info
MODSECURITY_CRS_PARANOIA_LEVEL=0
USE_BAD_BEHAVIOR=no
USE_MODSECURITY=no

# === Scheduler ===
BUNKERWEB_INSTANCES=bunkerweb
```

#### 2. Fichier `docker-compose.yml` (version actuelle)

```yaml
services:
  bunkerweb:
    image: bunkerity/bunkerweb:1.6.9
    container_name: bunkerweb
    ports:
      - "80:8080/tcp"
      - "443:8443/tcp"
      - "443:8443/udp"
    volumes:
      - bw-data:/data
      - ./www:/data/www:ro
    env_file:
      - .env
    dns:
      - 10.0.0.1
    networks:
      - bw-universe
      - bw-services
    restart: unless-stopped
    labels:
      - "bunkerweb.INSTANCE=yes"

  bw-scheduler:
    image: bunkerity/bunkerweb-scheduler:1.6.9
    container_name: bw-scheduler
    depends_on:
      bunkerweb:
        condition: service_started
    volumes:
      - bw-data:/data
    env_file:
      - .env
    dns:
      - 10.0.0.1
    networks:
      - bw-universe
      - bw-services
    restart: unless-stopped

  docker-socket-proxy:
    image: tecnativa/docker-socket-proxy:nightly
    container_name: docker-socket-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    dns:
      - 10.0.0.1
    networks:
      - bw-universe
    restart: unless-stopped

volumes:
  bw-data:

networks:
  bw-universe:
    name: bw-universe
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
  bw-services:
    name: bw-services
    driver: bridge
    ipam:
      config:
        - subnet: 172.29.0.0/16
```

### Points clés expliqués

- **`SERVE_FILES=yes`** + **`ROOT_FOLDER=/data/www`** → C’est **la vraie solution** (pas `STATIC_SITE`).  
- Montage `./www:/data/www:ro` → Le dossier local est bien lié au bon chemin dans le conteneur.  
- `bw-universe` ajouté sur le service `bunkerweb` → Résout les problèmes de communication scheduler ↔ bunkerweb.  
- Sous-réseaux IP fixes (IPAM) → Évite les changements d’IP Docker à chaque redémarrage.

### Vérification rapide

```bash
docker compose ps
docker exec bunkerweb ls -la /data/www/
```

**Capture d’écran actuelle** : Page avec les cœurs affichée correctement sur `http://10.0.10.x/index.html`.

![Capture d’écran](../images/Pasted%20image%2020260517050810.png)

### Explication détaillée des variables importantes

- **`SERVE_FILES=yes`** → Active le mode serveur de fichiers statiques (c’était la variable manquante depuis le début).
- **`ROOT_FOLDER=/data/www`** → Indique explicitement à BunkerWeb où se trouvent les fichiers HTML.
- **`./www:/data/www:ro`** → Monte ton dossier local `./www` vers le bon chemin dans le conteneur.
- **`DISABLE_DEFAULT_SERVER=no`** → Autorise le serveur par défaut pour servir la page statique.

### Vérification

```bash
# Vérifier que le fichier est bien dans le conteneur
docker exec bunkerweb ls -la /data/www/

# Logs récents
docker logs bunkerweb --tail 30
docker logs bw-scheduler --tail 20
```

---

**Document historique – Configuration BunkerWeb fonctionnelle (mai 2026)**  
*Version portfolio sanitisée – 15 août 2026*
