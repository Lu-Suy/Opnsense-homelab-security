# 21 – Multisite BunkerWeb (horus-ais.com + mrdoolux.horus-ais.com)

**Projet :** Bastion Godmode / OPNsense Homelab Security / Horus AIS  
**Date :** 31 juillet 2026 (soir) → formalisé 2 août 2026  
**Machine :** Prodesk (Debian 12) – bastion-godmode (`10.0.10.10`)  
**Objectif :** Activer le mode multisite de BunkerWeb de façon propre et isolée, sans casser le site existant, en restant derrière Cloudflare Tunnel (pas de Let’s Encrypt local).

---

## 1. Décision validée (domaines Web3)

| Domaine Web3       | Action                          | Cible |
|--------------------|----------------------------------|-------|
| `mrdoolux.brave`   | Redirection Web2 (à configurer plus tard) | `https://mrdoolux.horus-ais.com` |
| `lsdoolsd.brave`   | Reste en Web3 pur (IPFS plus tard) | — |

Les autres domaines Web3 restent en attente de décision.

---

## 2. État avant intervention

- `MULTISITE=no`
- `SERVER_NAME=10.0.10.10` (ou équivalent)
- Dossier web : `/opt/bunkerweb/www` monté en `/data/www` dans le conteneur
- Une seule page à la racine : `www/index.html` (page des cœurs)
- Anciennes configurations Web3 de mai 2026 commentées (obsolètes, non réutilisées)
- Cloudflare Tunnel ne connaissait que `horus-ais.com` et `www.horus-ais.com`

---

## 3. Architecture retenue (propre et minimale)

| Hostname                    | Dossier hôte                          | Dossier conteneur              | Contenu |
|----------------------------|---------------------------------------|--------------------------------|---------|
| `horus-ais.com`            | `/opt/bunkerweb/www/horus-ais`        | `/data/www/horus-ais`          | Page des cœurs (existante) |
| `mrdoolux.horus-ais.com`   | `/opt/bunkerweb/www/mrdoolux`         | `/data/www/mrdoolux`           | Page de test « MrDoolux » |

**Important :**  
Comme le trafic arrive via **Cloudflare Tunnel**, on n’utilise **pas** de certificats Let’s Encrypt côté BunkerWeb.  
Le tunnel termine le TLS. On pointe donc vers `http://localhost:80`.

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
        <p>Sous-domaine opérationnel – Bastion Godmode</p>
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
Exécute tout le bloc et envoie-moi la sortie des trois `ls`.
![capture](../images/Pasted%20image%2020260731164348.png)

### 4.3 Modification du `.env`

Ouvre le fichier :

```bash
cd /opt/bunkerweb
nano .env
```
#### 1. Active le multisite et définis les noms de serveurs
Changements clés :

```env
MULTISITE=yes
SERVER_NAME=horus-ais.com mrdoolux.horus-ais.com
DISABLE_DEFAULT_SERVER=yes
```
#### 2. Ajoute la configuration spécifique à chaque site (en bas du fichier)
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

#### 3. Sécurité (pour l’instant on laisse léger)

On **ne touche pas** encore à `USE_MODSECURITY` et `MODSECURITY_CRS_PARANOIA_LEVEL`.  
On les réactivera proprement une fois que le multisite fonctionne.

---

**Résumé de ce que tu fais dans nano :**
1. Change `MULTISITE=no` → `MULTISITE=yes`
2. Change la ligne `SERVER_NAME=...`
3. Mets `DISABLE_DEFAULT_SERVER=yes`
4. Ajoute les 4 lignes des `ROOT_FOLDER` à la fin
5. Sauvegarde : `Ctrl + O` → Entrée → `Ctrl + X`

### 4.4 Redémarrage BunkerWeb

```bash
cd /opt/bunkerweb
docker compose down
docker compose up -d
docker logs bunkerweb --tail 30
```

**Résultat observé :**  
Les logs confirment le chargement des deux services :
- `successfully loaded ... for the service: mrdoolux.horus-ais.com`
- `successfully loaded ... for the service: horus-ais.com`
- Message final : « BunkerWeb is ready to fool hackers ! »

(Un timeout DNSBL vers Spamhaus via `10.0.0.1` est apparu – non bloquant.)

### 4.5 Mise à jour du Cloudflare Tunnel

Actuellement le tunnel ne connaît que `horus-ais.com`.  
Il faut lui dire de aussi router `mrdoolux.horus-ais.com` vers BunkerWeb.

Regarde d’abord la config actuelle du tunnel :

```bash
cat /etc/cloudflared/config.yml
```

![capture](../images/Pasted%20image%2020260731170044.png)


Parfait. Voici la config actuelle.

On va simplement ajouter le nouveau hostname.

### Modification du fichier

```bash
sudo nano /etc/cloudflared/config.yml
```

Modifie le bloc `ingress` pour qu’il ressemble à ça :

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

Sauvegarde (`Ctrl+O` → Entrée → `Ctrl+X`).

Puis :

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared --no-pager
```
![capture](../images/Pasted%20image%2020260731170143.png)

**Pourquoi `http://localhost:80` et pas `https://...` ?**  
Le chiffrement est déjà terminé au bord Cloudflare.  
Le segment `cloudflared → BunkerWeb` reste entièrement local (127.0.0.1).  
Ajouter du TLS local n’apporte aucun gain de sécurité et complexifie la gestion des certificats.

Voici le chemin réel du trafic :

```
Visiteur → HTTPS (chiffré) → Cloudflare Edge
                ↓
         Cloudflare Tunnel (chiffré)
                ↓
         cloudflared (sur ta Prodesk)
                ↓
         http://localhost:80  ← ici c’est en local, sur la machine elle-même
                ↓
         BunkerWeb
```

**Points importants :**

1. **Entre le visiteur et Cloudflare** → tout est en HTTPS (TLS).
2. **Entre Cloudflare et ta machine** → le tunnel est aussi chiffré.
3. **Entre `cloudflared` et BunkerWeb** → ça passe par `localhost` (127.0.0.1).  
   Ça ne sort **jamais** sur le réseau. Personne d’extérieur ne peut intercepter ce trafic.

C’est pour ça que Cloudflare et la documentation officielle recommandent presque toujours `http://localhost:PORT` (ou `http://127.0.0.1:PORT`).

Mettre `https://localhost:443` obligerait BunkerWeb à avoir des certificats valides pour `mrdoolux.horus-ais.com` et `horus-ais.com`, ce qui est inutile et plus compliqué dans ce schéma.

### Et si le tunnel tombe ?

Si `cloudflared` s’arrête :
- Les sites deviennent inaccessibles depuis Internet (c’est normal).
- Personne ne peut « rentrer » par le tunnel, parce qu’il n’y a plus de tunnel.

### Est-ce que quelqu’un peut « rentrer dans le tunnel » ?

Non, pas de façon simple.  
Le tunnel est authentifié avec les credentials (`fa29b757-....json`) et la connexion est chiffrée.  
C’est justement pour ça qu’on a passé le service sous l’utilisateur `cloudflared` non-root et qu’on a mis les permissions strictes.


### 4.6 Enregistrement DNS

On va dire à Cloudflare que `mrdoolux.horus-ais.com` doit passer par ton tunnel.

Création manuelle du CNAME dans le dashboard Cloudflare (zone `horus-ais.com`) :
1. Va dans le dashboard Cloudflare classique (pas Zero Trust) :  
   [https://dash.cloudflare.com](https://dash.cloudflare.com)

2. Sélectionne le domaine **horus-ais.com**

3. Va dans **DNS** → **Records**

4. Clique sur **Add record**

Remplis ainsi :

| Champ          | Valeur                                              |
|----------------|-----------------------------------------------------|
| Type           | CNAME                                               |
| Name           | `mrdoolux`                                          |
| Target         | `fa29b757-5fb8-4867-ab4e-86970619be0d.cfargotunnel.com` |
| Proxy status   | Proxied (orange)                                    |
| TTL            | Auto                                                |

(Commande alternative possible : `cloudflared tunnel route dns ...`)
La méthode la plus simple depuis la machine :

```bash
sudo cloudflared tunnel route dns fa29b757-5fb8-4867-ab4e-86970619be0d mrdoolux.horus-ais.com
```

## 5. Tests de validation

| Test                                      | Résultat                  | Commentaire |
|-------------------------------------------|---------------------------|-----------|
| `curl -I https://horus-ais.com`           | HTTP 200                  | Page des cœurs intacte |
| `curl -I https://mrdoolux.horus-ais.com`  | HTTP 200 + HTML           | Page de test servie correctement |
| Navigateur (cache DNS)                    | Erreur temporaire puis OK | Résolu par navigation privée / attente / 4G |
| Logs BunkerWeb                            | Chargement des 2 sites    | OK |
| Service `cloudflared`                     | 4 connexions Registered   | OK |


![capture](../images/Pasted%20image%2020260731173026.png)
et dans le navigateur.
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

1. **Contenu réel** des sites (remplacer la page de test mrdoolux + enrichir horus-ais.com)
2. Redirection Unstoppable `mrdoolux.brave` → `https://mrdoolux.horus-ais.com`
3. Ajout d’éventuels autres sous-domaines (même méthode)
4. Réactivation progressive de ModSecurity / règles plus strictes une fois le multisite stable
5. WireGuard / Wazuh / LUKS selon priorités du document 18A

---

## 8. Commandes de vérification rapides (à garder)

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
