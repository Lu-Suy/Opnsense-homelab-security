# 21 – Multisite BunkerWeb (horus-ais.com + sous-domaines)

**Date originale :** 31 juillet 2026 (soir) → formalisé 2 août 2026  
**Machine :** Prodesk (Debian 12) – bastion  
**Version portfolio :** 15 août 2026 (sanitisée)  
**Objectif :** Activer le mode multisite de BunkerWeb de façon propre et isolée, sans casser le site existant, en restant derrière Cloudflare Tunnel (pas de Let’s Encrypt local).

---

## Pourquoi le mode multisite ?

BunkerWeb en mode mono-site ne gère qu’un seul hostname.  
Dès qu’on veut servir plusieurs sites (ou sous-domaines) avec des racines de fichiers isolées, il faut activer le mode **MULTISITE**.

**Avantages :**
- Isolation des contenus (chaque hostname a son propre dossier)
- Pas de partage involontaire de fichiers entre sites
- Configuration claire et maintenable
- Compatible avec Cloudflare Tunnel (TLS terminé au bord)

**Contrainte importante :**  
Comme le trafic arrive via Cloudflare Tunnel, on n’utilise **pas** de certificats Let’s Encrypt côté BunkerWeb.  
Le tunnel termine le TLS. On pointe donc vers `http://localhost:80`.

---

## 1. Décision validée (domaines Web3)

| Domaine Web3       | Action                          | Cible |
|--------------------|----------------------------------|-------|
| `mrdoolux.brave`   | Redirection Web2 (à configurer plus tard) | `https://mrdoolux.horus-ais.com` |
| Autres domaines Web3 | En attente de décision          | — |

---

## 2. État avant intervention

- `MULTISITE=no`
- Une seule page à la racine (`www/index.html`)
- Cloudflare Tunnel ne connaissait que `horus-ais.com` et `www.horus-ais.com`

---

## 3. Architecture retenue (propre et minimale)

| Hostname                    | Dossier hôte                          | Dossier conteneur              | Contenu |
|----------------------------|---------------------------------------|--------------------------------|---------|
| `horus-ais.com`            | `/opt/bunkerweb/www/horus-ais`        | `/data/www/horus-ais`          | Page principale |
| `mrdoolux.horus-ais.com`   | `/opt/bunkerweb/www/mrdoolux`         | `/data/www/mrdoolux`           | Page de test |

---

## 4. Étapes réalisées

### 4.1 Structure de dossiers

```bash
cd /opt/bunkerweb
mkdir -p www/horus-ais www/mrdoolux
mv www/index.html www/horus-ais/index.html
```

### 4.2 Page de test mrdoolux

```bash
cat > www/mrdoolux/index.html << 'EOF'
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>MrDoolux – Horus AIS</title>
    <style>
        body { font-family: system-ui, sans-serif; background: #0f0f0f; color: #eee; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
        h1 { font-size: 2.5rem; }
        p { color: #aaa; }
    </style>
</head>
<body>
    <div style="text-align: center;">
        <h1>MrDoolux</h1>
        <p>Sous-domaine opérationnel</p>
        <p style="font-size: 0.9rem; margin-top: 2rem;">mrdoolux.horus-ais.com</p>
    </div>
</body>
</html>
EOF

# Vérification
ls -la www/
ls -la www/horus-ais/
ls -la www/mrdoolux/
```

![capture](../images/Pasted%20image%2020260731164348.png)

### 4.3 Modification du `.env`

```bash
cd /opt/bunkerweb
nano .env
```

Changements clés :

```env
MULTISITE=yes
SERVER_NAME=horus-ais.com mrdoolux.horus-ais.com
DISABLE_DEFAULT_SERVER=yes
```

Ajout en bas du fichier :

```env
# === Site principal ===
horus-ais.com_ROOT_FOLDER=/data/www/horus-ais
horus-ais.com_SERVE_FILES=yes

# === Sous-domaine MrDoolux ===
mrdoolux.horus-ais.com_ROOT_FOLDER=/data/www/mrdoolux
mrdoolux.horus-ais.com_SERVE_FILES=yes
```

**Effet :**  
BunkerWeb charge deux virtual hosts isolés. Chaque hostname a son propre dossier racine.  
Le mode `DISABLE_DEFAULT_SERVER=yes` empêche de servir un site « par défaut » sur une IP nue.

### 4.4 Redémarrage BunkerWeb

```bash
cd /opt/bunkerweb
docker compose down
docker compose up -d
docker logs bunkerweb --tail 30
```

**Résultat observé :**  
Les logs confirment le chargement des deux services.

### 4.5 Mise à jour du Cloudflare Tunnel

```bash
sudo nano /etc/cloudflared/config.yml
```

Bloc `ingress` final :

```yaml
ingress:
  - hostname: horus-ais.com
    service: http://localhost:80
  - hostname: www.horus-ais.com
    service: http://localhost:80
  - hostname: mrdoolux.horus-ais.com
    service: http://localhost:80
  - service: http_status:404
```

**Important :**  
La règle `service: http_status:404` doit **toujours** rester en **dernier**.

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared --no-pager
```

![capture](../images/Pasted%20image%2020260731170143.png)

**Pourquoi `http://localhost:80` et pas `https://...` ?**  

Le chiffrement est déjà terminé au bord Cloudflare.  
Le segment `cloudflared → BunkerWeb` reste entièrement local (127.0.0.1).  
Ajouter du TLS local n’apporte aucun gain de sécurité et complexifie la gestion des certificats.

Chemin réel du trafic :

```
Visiteur → HTTPS (chiffré) → Cloudflare Edge
                ↓
         Cloudflare Tunnel (chiffré)
                ↓
         cloudflared (sur le bastion)
                ↓
         http://localhost:80  ← local uniquement
                ↓
         BunkerWeb
```

### 4.6 Enregistrement DNS

Création du CNAME dans le dashboard Cloudflare (zone `horus-ais.com`) :

| Champ          | Valeur                                              |
|----------------|-----------------------------------------------------|
| Type           | CNAME                                               |
| Name           | `mrdoolux`                                          |
| Target         | `<TUNNEL-ID>.cfargotunnel.com`                      |
| Proxy status   | Proxied (orange)                                    |
| TTL            | Auto                                                |

Commande alternative :

```bash
sudo cloudflared tunnel route dns <TUNNEL-ID> mrdoolux.horus-ais.com
```

---

## 5. Tests de validation

| Test                                      | Résultat                  | Commentaire |
|-------------------------------------------|---------------------------|-----------|
| `curl -I https://horus-ais.com`           | HTTP 200                  | Page principale intacte |
| `curl -I https://mrdoolux.horus-ais.com`  | HTTP 200 + HTML           | Page de test servie correctement |
| Logs BunkerWeb                            | Chargement des 2 sites    | OK |
| Service `cloudflared`                     | Connexions Registered     | OK |

![capture](../images/Pasted%20image%2020260731173026.png)
![capture](../images/Screenshot%202026-08-03%20123521.png)

**Conclusion :** Multisite **opérationnel**.

---

## 6. Points de sécurité / bonnes pratiques respectés

- Moindre privilège déjà en place sur `cloudflared` (user non-root – doc 20)
- Isolation des racines de sites (pas de partage de fichiers entre hostnames)
- Pas d’exposition de ports supplémentaires
- Pas de certificats locaux inutiles
- `DISABLE_DEFAULT_SERVER=yes` → pas de site par défaut sur IP

---

## 7. Suite logique

1. **Contenu réel** des sites
2. Redirection Unstoppable `mrdoolux.brave` → `https://mrdoolux.horus-ais.com`
3. Ajout d’éventuels autres sous-domaines (même méthode)
4. Réactivation progressive de ModSecurity / règles plus strictes
5. WireGuard / Wazuh / LUKS selon priorités

---

## 8. Commandes de vérification rapides

```bash
# Structure
ls -la /opt/bunkerweb/www/
ls -la /opt/bunkerweb/www/horus-ais/
ls -la /opt/bunkerweb/www/mrdoolux/

# Conteneurs
docker ps --format "table {{.Names}}\t{{.Status}}"

# Logs BunkerWeb
docker logs bunkerweb --tail 50

# Config tunnel
cat /etc/cloudflared/config.yml

# Test rapide
curl -I https://horus-ais.com
curl -I https://mrdoolux.horus-ais.com
```

---

**Document formalisé le 2 août 2026**  
**Statut :** Multisite BunkerWeb **terminé et validé**.

**Document de référence – Multisite BunkerWeb**  
*Version portfolio sanitisée – 15 août 2026*
