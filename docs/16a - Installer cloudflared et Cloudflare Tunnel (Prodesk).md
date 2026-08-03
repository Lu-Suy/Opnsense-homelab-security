# 16 – Installer cloudflared et Cloudflare Tunnel (Prodesk)

**Projet :** Bastion Godmode / OPNsense Homelab Security  
**Date :** 30 juillet 2026  
**Machine :** Prodesk (Debian 12) – `10.0.10.10` / `10.0.10.11`  
**Objectif :** Exposer BunkerWeb (`horus-ais.com`) de façon sécurisée via Cloudflare Tunnel, sans ouvrir de ports sur la box Nordnet / OPNsense.

---

## 1. Résumé de l’architecture finale

| Élément | Valeur |
|---------|--------|
| Domaine | `horus-ais.com` + `www.horus-ais.com` |
| Tunnel name | `bastion-godmode` |
| Tunnel ID | `fa29b757-5fb8-4867-ab4e-86970619be0d` |
| Protocole | QUIC (port 7844) |
| Service local | `http://localhost:80` (BunkerWeb) |
| Service systemd | `cloudflared.service` (enabled + running) |
| Config système | `/etc/cloudflared/config.yml` |
| Credentials | `/etc/cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json` |

**Résultat :**  
`https://horus-ais.com` affiche la page Bastion Godmode protégée par BunkerWeb + Cloudflare (IP réelle cachée, aucun port ouvert).

---

## 2. Prérequis réalisés

- Domaine `horus-ais.com` ajouté dans Cloudflare (plan Free)
- Nameservers changés chez Unstoppable Domains → Cloudflare (`armando.ns.cloudflare.com` / `sonia.ns.cloudflare.com`)
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
# → cloudflared version 2026.7.3
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
2. Ouvre-le dans ton navigateur (sur l’AlphaDeck)
3. Connecte-toi avec ton compte Cloudflare si besoin
4. Sélectionne le domaine **horus-ais.com**
5. Autorise

Une fois que c’est fait, tu verras un message de succès dans le terminal.
![capture](../images/Pasted%20image%2020260730144108.png)
![capture](../images/Pasted%20image%2020260730145008.png)
![capture](../images/Pasted%20image%2020260730145102.png)

– Créer le tunnel

Lance cette commande :
```bash
cloudflared tunnel create bastion-godmode
# → Tunnel ID : fa29b757-5fb8-4867-ab4e-86970619be0d
# → Credentials : ~/.cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json
```
Elle va créer un tunnel nommé `bastion-godmode` et te donner un **Tunnel ID** (une longue chaîne de caractères).
![capture](../images/Pasted%20image%2020260730145414.png)


## 5. Fichier de configuration

On va créer le fichier de config du tunnel.

Exécute ces commandes :

```bash
mkdir -p ~/.cloudflared
```

```bash
nano ~/.cloudflared/config.yml
```

Dans le fichier `nano`, colle exactement ceci :

```yaml
tunnel: fa29b757-5fb8-4867-ab4e-86970619be0d
credentials-file: /etc/cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json

ingress:
  - hostname: horus-ais.com
    service: http://localhost:80
  - hostname: www.horus-ais.com
    service: http://localhost:80
  - service: http_status:404
```
**Sauvegarde** :
- `Ctrl + O` → Entrée
- `Ctrl + X`
---

## 6. Routes DNS

Il y a encore l’ancien record **A** (celui qui pointait vers IP_PUBLIC) qui bloque la création du CNAME.

### Ce qu’il faut faire

1. Va dans le dashboard Cloudflare → **horus-ais.com** → **DNS** → **Records**
2. Tu vas voir un record de type **A** pour horus-ais.com (ou @)
3. **Supprime-le** (bouton Delete)
4. S’il y a aussi un record pour www, supprime-le aussi s’il n’est pas un CNAME vers le tunnel

On va dire à Cloudflare que `horus-ais.com` et `www.horus-ais.com` doivent pointer vers notre tunnel.

```bash
cloudflared tunnel route dns bastion-godmode horus-ais.com
cloudflared tunnel route dns bastion-godmode www.horus-ais.com
```
![capture](../images/Pasted%20image%2020260730153404.png)
---

## 7. Règle firewall OPNsense (critique)

1. Va dans **Firewall → Rules → OPT1** (le réseau de la Prodesk)
2. Clique sur **Add** (en haut à droite ou en bas)
3. Configure comme suit :

| Champ                | Valeur                                     |
| -------------------- | ------------------------------------------ |
| **Action**           | Pass                                       |
| **Interface**        | OPT1                                       |
| **Direction**        | in                                         |
| **Protocol**         | UDP (ou TCP/UDP si possible)               |
| **Source**           | OPT1 net                                   |
| **Destination**      | any (ou Cloudflare si tu veux être strict) |
| **Destination Port** | 7844                                       |
| **Description**      | Allow Cloudflare Tunnel (QUIC)             |

4. Sauvegarde et **Apply Changes**

> Note : L’alias `Cloudflare_IPs` (URL Table) a été créé mais n’a pas fonctionné immédiatement. On a utilisé `any` pour débloquer. À retravailler plus tard.

---
on peut créer un **Alias**.

Voici comment faire :

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

### Étape 8 – Lancer le tunnel

On va d’abord le tester en avant-plan pour vérifier que tout fonctionne :

```bash
cloudflared tunnel run bastion-godmode
```

Laisse cette commande tourner et regarde s’il y a des erreurs.

Ensuite, ouvre ton navigateur (sur l’AlphaDeck) et va sur :

- https://horus-ais.com
- https://www.horus-ais.com

![capture](../images/Pasted%20image%2020260730162932.png)

### a. Est-ce que la page est vulnérable ?

**Beaucoup moins qu’un site classique exposé.**

Voici pourquoi :

| Protection | Statut |
|------------|--------|
| IP réelle de ta box **cachée** | Oui (Cloudflare) |
| Aucun port ouvert sur la Nordnet / OPNsense | Oui |
| Trafic passe par le réseau Cloudflare | Oui |
| BunkerWeb (WAF) devant | Oui |
| HTTPS géré par Cloudflare | Oui |

Ce n’est **pas invulnérable** (rien ne l’est), mais c’est une architecture **très solide** pour un lab / petit site.

Les risques principaux restants sont :
- Une faille dans BunkerWeb lui-même
- Une mauvaise configuration des règles BunkerWeb
- Une attaque applicative sur le contenu que tu mettras derrière

Mais l’exposition réseau est très bien maîtrisée.

### b. Pourquoi les moteurs de recherche ne le trouvent pas ?

C’est **normal** pour l’instant.

- Le site vient juste d’être mis en ligne
- Il n’y a quasiment aucun contenu / lien externe qui pointe vers lui
- Google / Bing mettent du temps à découvrir et indexer un nouveau domaine (parfois plusieurs jours à plusieurs semaines)
- Tu n’as pas encore soumis le sitemap ni fait de Search Console

Pour qu’il apparaisse, il faudra plus tard :
- Du contenu
- Des liens
- Éventuellement Google Search Console

Ce n’est **pas un problème de sécurité**, c’est juste qu’il est tout neuf.

---

## 9. Service systemd

### Installation


`cloudflared service install` cherche le fichier de config dans les emplacements standards, et le nôtre est dans ton home.

### Solution : 

On va copier la config au bon endroit :

```bash
# 1. Créer le dossier système
sudo mkdir -p /etc/cloudflared
```

```bash
# 2. Copier le fichier de configuration
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
```

```bash
# 3. Copier aussi les credentials (important)
sudo cp ~/.cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json /etc/cloudflared/
```

```bash
# 4. Adapter le chemin dans le config système
sudo nano /etc/cloudflared/config.yml
```

Dans ce fichier, change la ligne `credentials-file` pour :

```yaml
credentials-file: /etc/cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json
```

Sauvegarde (`Ctrl+O` → Entrée → `Ctrl+X`).
---

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
---
Comment ça fonctionne maintenant

| Élément                                                     | Explication                                                                                                                                            |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Au démarrage de la Prodesk**                              | Le service `cloudflared` démarre automatiquement grâce à `systemctl enable`. Il lit le fichier `/etc/cloudflared/config.yml` et se connecte au tunnel. |
| **Dans Cloudflare**                                         | Rien à faire de spécial. Le tunnel reste le même. Tant que le service tourne, Cloudflare voit la connexion.                                            |
| **Si tu arrêtes le service** (`systemctl stop cloudflared`) | Le tunnel se déconnecte. Le site devient inaccessible de l’extérieur.                                                                                  |
| **Si tu redémarres le service**                             | Il se reconnecte avec **la même** identité de tunnel. La clé ne change pas.                                                                            |
| **Si tu redémarres la machine**                             | Le service repart tout seul et se reconnecte.                                                                                                          |
---

## 10. Sécurité des credentials

Les fichiers sensibles sont en clair sur le disque :

```bash
/etc/cloudflared/config.yml
/etc/cloudflared/fa29b757-5fb8-4867-ab4e-86970619be0d.json
```
Et aussi (copie d’origine) :

```bash
/home/hyper_doo/.cloudflared/
```

Le fichier `.json` contient les credentials qui permettent à `cloudflared` de s’authentifier auprès de Cloudflare.  
C’est **confidentiel** : ne le partage jamais et ne le mets pas dans un dépôt Git public.

Le certificat d’origine (`cert.pem`) sert surtout pour créer/gérer les tunnels. Une fois le tunnel créé, c’est surtout le fichier `.json` qui est utilisé au quotidien.

### En résumé

- Tu n’as **rien** à faire dans le dashboard Cloudflare à chaque redémarrage.
- La clé du tunnel **ne change pas** quand tu stoppes/redémarres le service.
- Tout est maintenant persistant et automatique.

### Est-ce une pratique courante ?

Oui, c’est courant.  
La protection repose principalement sur :

1. **Restreindre l’accès à la machine** (SSH durci, Fail2ban, pas d’accès root direct, etc.)
2. **Permissions strictes** sur les fichiers de credentials
3. **Chiffrement du disque** (LUKS) → très bon point que tu soulèves
4. Eventuellement des solutions plus avancées (TPM, secrets managers, etc.)
     
**Permissions renforcées appliquées :**
```bash
sudo chmod 600 /etc/cloudflared/*.json
sudo chmod 600 /etc/cloudflared/config.yml
sudo chown root:root /etc/cloudflared/*
```
Comme ça, seul root peut les lire.


**Recommandation future :** Réinstallation de Debian avec LUKS (voir fichier **17**).

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
- [ ] Créer un utilisateur dédié non-root pour faire tourner cloudflared (durcissement)
- [ ] Documenter les utilisateurs existants de la Prodesk (dans fichier 05 ou architecture)
- [ ] Réinstallation Debian + LUKS (checklist fichier 17)
- [ ] Indexation Google / Search Console (plus tard)

---

## 13. Liens utiles

- Dashboard Cloudflare : https://dash.cloudflare.com
- Tunnel ID : `fa29b757-5fb8-4867-ab4e-86970619be0d`
- Documentation Cloudflare Tunnel : https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/

---

**Document créé le 30 juillet 2026**  
**Statut :** Tunnel opérationnel et en service systemd.
