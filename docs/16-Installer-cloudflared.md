# 16 – Installer cloudflared et Cloudflare Tunnel

**Date originale :** 30 juillet 2026  
**Machine :** Prodesk (Debian 12) – zone OPT1  
**Version portfolio :** 15 août 2026 (sanitisée)  
**Objectif :** Exposer BunkerWeb (`horus-ais.com`) de façon sécurisée via Cloudflare Tunnel, sans ouvrir de ports sur la box FAI ni sur OPNsense.

---

## Pourquoi cette installation ?

Après avoir basculé le DNS vers Cloudflare (document 15), il reste à créer le **tunnel** qui relie le bastion local au réseau Cloudflare.

`cloudflared` est le client officiel qui :
- Établit une connexion **sortante** (outbound) vers Cloudflare
- Transmet le trafic HTTPS reçu par Cloudflare vers le service local (BunkerWeb sur le port 80)
- Évite complètement d’ouvrir les ports 80/443 sur le WAN

C’est le cœur de l’architecture « zero open ports ».

---

## 1. Résumé de l’architecture finale

| Élément | Valeur |
|---------|--------|
| Domaine | `horus-ais.com` + `www.horus-ais.com` |
| Tunnel name | `bastion-tunnel` (exemple) |
| Protocole | QUIC (port 7844) |
| Service local | `http://localhost:80` (BunkerWeb) |
| Service systemd | `cloudflared.service` (enabled + running) |
| Config système | `/etc/cloudflared/config.yml` |
| Credentials | `/etc/cloudflared/<TUNNEL-ID>.json` |

**Résultat :**  
`https://horus-ais.com` affiche la page protégée par BunkerWeb + Cloudflare (IP réelle cachée, aucun port ouvert).

---

## 2. Prérequis réalisés

- Domaine `horus-ais.com` ajouté dans Cloudflare (plan Free)
- Nameservers changés chez Unstoppable Domains → Cloudflare
- Domaine passé en statut **Active**
- Ancien record A (IP publique) supprimé
- CNAME créés via `cloudflared tunnel route dns`

---

## 3. Installation de cloudflared

Exécute ces commandes **une par une** :

```bash
# Téléchargement
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Installation
sudo dpkg -i cloudflared.deb

# Vérification
cloudflared --version
# → cloudflared version 2026.7.3 (ou plus récente)
```

---

## 4. Authentification & création du tunnel

Lance cette commande :

```bash
cloudflared tunnel login
# → certificat enregistré dans ~/.cloudflared/cert.pem
```

Elle va t’afficher un **lien** (ou un QR code selon les cas).

1. Copie le lien
2. Ouvre-le dans ton navigateur (depuis la workstation d’administration)
3. Connecte-toi avec ton compte Cloudflare si besoin
4. Sélectionne le domaine **horus-ais.com**
5. Autorise

Une fois que c’est fait, tu verras un message de succès dans le terminal.

![capture](../images/Pasted%20image%2020260730144108.png)
![capture](../images/Pasted%20image%2020260730145008.png)
![capture](../images/Pasted%20image%2020260730145102.png)

### Créer le tunnel

```bash
cloudflared tunnel create bastion-tunnel
# → Tunnel ID : <longue-chaîne>
# → Credentials : ~/.cloudflared/<TUNNEL-ID>.json
```

Elle va créer un tunnel nommé `bastion-tunnel` et te donner un **Tunnel ID**.

![capture](../images/Pasted%20image%2020260730145414.png)

---

## 5. Fichier de configuration

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

Dans le fichier, colle (en adaptant le Tunnel ID) :

```yaml
tunnel: <TUNNEL-ID>
credentials-file: /etc/cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: horus-ais.com
    service: http://localhost:80
  - hostname: www.horus-ais.com
    service: http://localhost:80
  - service: http_status:404
```

**Sauvegarde** : `Ctrl + O` → Entrée → `Ctrl + X`

---

## 6. Routes DNS

Il y a encore l’ancien record **A** (celui qui pointait vers l’IP publique) qui bloque la création du CNAME.

### Ce qu’il faut faire

1. Va dans le dashboard Cloudflare → **horus-ais.com** → **DNS** → **Records**
2. Tu vas voir un record de type **A** pour horus-ais.com (ou @)
3. **Supprime-le**
4. S’il y a aussi un record pour www, supprime-le aussi s’il n’est pas un CNAME vers le tunnel

Ensuite :

```bash
cloudflared tunnel route dns bastion-tunnel horus-ais.com
cloudflared tunnel route dns bastion-tunnel www.horus-ais.com
```

![capture](../images/Pasted%20image%2020260730153404.png)

---

## 7. Règle firewall OPNsense (critique)

Le tunnel utilise le protocole **QUIC** (UDP 7844) en **sortie**.  
Il faut autoriser ce trafic depuis la zone du bastion.

1. Va dans **Firewall → Rules → OPT1**
2. Clique sur **Add**
3. Configure :

| Champ                | Valeur                                     |
| -------------------- | ------------------------------------------ |
| **Action**           | Pass                                       |
| **Interface**        | OPT1                                       |
| **Direction**        | in                                         |
| **Protocol**         | UDP (ou TCP/UDP si possible)               |
| **Source**           | OPT1 net                                   |
| **Destination**      | any (ou alias Cloudflare_IPs plus tard)    |
| **Destination Port** | 7844                                       |
| **Description**      | Allow Cloudflare Tunnel (QUIC)             |

4. Sauvegarde et **Apply Changes**

> Note : L’alias `Cloudflare_IPs` (URL Table) a été créé mais n’a pas fonctionné immédiatement. On a utilisé `any` pour débloquer. À retravailler plus tard.

### Création de l’alias (préparation)

1. Va dans **Firewall → Aliases**
2. Clique sur **+** (Add)
3. Configure :

| Champ           | Valeur                            |
| --------------- | --------------------------------- |
| **Name**        | Cloudflare_IPs                    |
| **Type**        | URL Table (IPs)                   |
| **Content**     | https://www.cloudflare.com/ips-v4 |
| **Description** | Cloudflare IPv4 ranges            |

![capture](../images/Pasted%20image%2020260730155336.png)

---

## 8. Lancer le tunnel (test)

On le teste d’abord en avant-plan :

```bash
cloudflared tunnel run bastion-tunnel
```

Laisse cette commande tourner et regarde s’il y a des erreurs.

Ensuite, ouvre ton navigateur et va sur :

- https://horus-ais.com
- https://www.horus-ais.com

![capture](../images/Pasted%20image%2020260730162932.png)

### Est-ce que la page est vulnérable ?

**Beaucoup moins qu’un site classique exposé.**

| Protection | Statut |
|------------|--------|
| IP réelle de la box **cachée** | Oui (Cloudflare) |
| Aucun port ouvert sur la box / OPNsense | Oui |
| Trafic passe par le réseau Cloudflare | Oui |
| BunkerWeb (WAF) devant | Oui |
| HTTPS géré par Cloudflare | Oui |

Ce n’est **pas invulnérable** (rien ne l’est), mais c’est une architecture **très solide** pour un lab / petit site.

Les risques principaux restants sont :
- Une faille dans BunkerWeb lui-même
- Une mauvaise configuration des règles BunkerWeb
- Une attaque applicative sur le contenu

Mais l’exposition réseau est très bien maîtrisée.

### Pourquoi les moteurs de recherche ne le trouvent pas ?

C’est **normal** pour l’instant.

- Le site vient juste d’être mis en ligne
- Il n’y a quasiment aucun contenu / lien externe
- Google / Bing mettent du temps à découvrir et indexer un nouveau domaine
- Tu n’as pas encore soumis de sitemap ni utilisé Search Console

Ce n’est **pas un problème de sécurité**, c’est juste qu’il est tout neuf.

---

## 9. Service systemd

`cloudflared service install` cherche le fichier de config dans les emplacements standards.  
On va donc placer la config au bon endroit :

```bash
# 1. Créer le dossier système
sudo mkdir -p /etc/cloudflared

# 2. Copier le fichier de configuration
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml

# 3. Copier aussi les credentials (important)
sudo cp ~/.cloudflared/<TUNNEL-ID>.json /etc/cloudflared/

# 4. Adapter le chemin dans le config système
sudo nano /etc/cloudflared/config.yml
```

Dans ce fichier, change la ligne `credentials-file` pour :

```yaml
credentials-file: /etc/cloudflared/<TUNNEL-ID>.json
```

### Correction timeout (obligatoire)

```bash
sudo systemctl edit cloudflared --full
```

Dans la section `[Service]` :

```ini
Type=simple
TimeoutStartSec=0
Restart=always
RestartSec=5
```

Puis :

```bash
sudo systemctl daemon-reload
sudo systemctl enable cloudflared
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
# → active (running)
```

![capture](../images/Pasted%20image%2020260730171215.png)

### Comment ça fonctionne maintenant

| Élément | Explication |
|---------|-------------|
| **Au démarrage de la machine** | Le service `cloudflared` démarre automatiquement et se connecte au tunnel |
| **Dans Cloudflare** | Rien à faire. Tant que le service tourne, Cloudflare voit la connexion |
| **Si tu arrêtes le service** | Le tunnel se déconnecte → site inaccessible de l’extérieur |
| **Si tu redémarres le service** | Il se reconnecte avec **la même** identité de tunnel |
| **Si tu redémarres la machine** | Le service repart tout seul |

---

## 10. Sécurité des credentials

Les fichiers sensibles sont en clair sur le disque :

```bash
/etc/cloudflared/config.yml
/etc/cloudflared/<TUNNEL-ID>.json
```

Le fichier `.json` contient les credentials qui permettent à `cloudflared` de s’authentifier auprès de Cloudflare.  
C’est **confidentiel** : ne le partage jamais et ne le mets pas dans un dépôt Git public.

### Permissions renforcées appliquées

```bash
sudo chmod 600 /etc/cloudflared/*.json
sudo chmod 600 /etc/cloudflared/config.yml
sudo chown root:root /etc/cloudflared/*
```

Comme ça, seul root peut les lire.

**Recommandation future :**  
- Utilisateur dédié non-root (voir document 20)  
- Chiffrement du disque (LUKS – document 17)

---

## 11. Commandes utiles

```bash
# Statut du service
sudo systemctl status cloudflared

# Logs
sudo journalctl -u cloudflared -f

# Redémarrer le tunnel
sudo systemctl restart cloudflared

# Arrêter
sudo systemctl stop cloudflared
```

---

## 12. Points restants / à améliorer

- [ ] Faire fonctionner l’alias `Cloudflare_IPs` correctement (au lieu de `any`)
- [ ] Créer un utilisateur dédié non-root pour faire tourner cloudflared (document 20)
- [ ] Réinstallation + LUKS (checklist document 17)
- [ ] Indexation Google / Search Console (plus tard)

---

## 13. Liens utiles

- Dashboard Cloudflare : https://dash.cloudflare.com
- Documentation Cloudflare Tunnel : https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/

---

**Document créé le 30 juillet 2026**  
**Statut :** Tunnel opérationnel et en service systemd.

**Document de référence – Installation cloudflared + Cloudflare Tunnel**  
*Version portfolio sanitisée – 15 août 2026*
