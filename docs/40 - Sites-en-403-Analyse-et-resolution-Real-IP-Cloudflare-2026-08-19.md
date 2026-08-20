# 40 – Sites en 403 : Analyse, résolution et configuration Real IP (Cloudflare Tunnel)

**Projet :** Bastion Godmode / Horus AIS  
**Date :** 19 août 2026  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Composants concernés :** BunkerWeb 1.6.13 + Cloudflare Tunnel + Docker  

**Objectif de cette fiche :**  
Documenter l’incident de blocage 403 sur les sites publics, l’analyse des causes, la résolution immédiate et la correction structurelle (transmission de la vraie IP client) afin d’éviter que le problème ne se reproduise.

---

## 1. Contexte et symptômes

Les sites publics étaient inaccessibles :

| Domaine                        | Comportement observé      |
|--------------------------------|---------------------------|
| `https://horus-ais.com`        | **403 Forbidden**         |
| `https://mrdoolux.horus-ais.com` | **403 Forbidden**       |

Les conteneurs BunkerWeb étaient pourtant **Up** et **healthy**.  
Le tunnel Cloudflare et la configuration DNS étaient corrects.

---

## 2. Investigation

### 2.1 Première analyse des logs

Commande utilisée :

```bash
docker logs --tail 100 bunkerweb | grep -iE "ban|403|bad.?behavior|denied|blocked"
```

**Observations clés extraites des logs :**

- Présence répétée de la raison : `client was denied with reason bad behavior`
- L’adresse IP source identifiée était systématiquement **`172.29.0.1`**
- Multiples tentatives sur des chemins typiques de scanners :
  - `/wp-admin/install.php`
  - `/robots.txt`
  - `/favicon.ico`
  - `/wp-includes/wlwmanifest.xml`
- Ban actif avec durée restante élevée (environ 13 heures)

### 2.2 Identification de l’IP

L’adresse `172.29.0.1` correspond à une IP du réseau Docker interne (`bw-services` – subnet `172.29.0.0/16`).  
Il s’agit de l’adresse vue par BunkerWeb pour **tout** le trafic provenant du Cloudflare Tunnel (cloudflared).

**Conclusion intermédiaire :**  
BunkerWeb ne voyait pas les vraies adresses IP des clients, mais uniquement l’IP interne du tunnel.  
Tous les scanners et les visiteurs légitimes partageaient la même adresse source → le plugin **Bad Behavior** a fini par bannir cette IP globale.

---

## 3. Cause racine

### 3.1 Mécanisme du ban

Le plugin **Bad Behavior** de BunkerWeb a détecté un comportement suspect (trop de 404 / chemins de scan WordPress en peu de temps) et a appliqué un ban de 24 heures sur l’IP `172.29.0.1`.

### 3.2 Pourquoi l’IP du tunnel a été bannie

- Cloudflare Tunnel injecte les headers `CF-Connecting-IP` et `X-Forwarded-For`
- BunkerWeb n’était **pas configuré** pour lire ces headers (`USE_REAL_IP` était désactivé)
- Par conséquent, BunkerWeb utilisait l’adresse IP de la connexion entrante (réseau Docker) comme adresse client

Ce comportement est classique lorsque BunkerWeb est placé derrière un reverse-proxy ou un tunnel sans configuration Real IP.

---

## 4. Résolution immédiate

### 4.1 Levée du ban

Commandes exécutées :

```bash
docker exec bunkerweb bwcli bans list
docker exec bunkerweb bwcli unban 172.29.0.1
```

Résultat :

```text
✅ SUCCESS
🔓 IP 172.29.0.1 has been unbanned globally
```

![[Pasted image 20260819091938.png]]

### 4.2 Vérification immédiate

Après le unban, les deux sites sont redevenus accessibles (HTTP 200) depuis l’extérieur.

---

## 5. Correction structurelle – Configuration Real IP

Pour éviter que le problème ne se reproduise, BunkerWeb a été configuré pour utiliser la **vraie adresse IP client** fournie par Cloudflare.

### 5.1 Localisation de la configuration

Les fichiers de configuration BunkerWeb se trouvent dans `/opt/bunkerweb/` :

```bash
ls -la /opt/bunkerweb/
```

Le fichier principal est `.env` (chargé par `docker-compose.yml` via `env_file`).

![[Pasted image 20260819093556.png]]

### 5.2 Variables ajoutées dans `/opt/bunkerweb/.env`

```bash
# === Real IP (Cloudflare Tunnel) ===
USE_REAL_IP=yes
REAL_IP_HEADER=CF-Connecting-IP
REAL_IP_FROM=172.16.0.0/12 172.28.0.0/16 172.29.0.0/16 10.0.0.0/8 192.168.0.0/16
REAL_IP_RECURSIVE=yes
```

**Explication des paramètres :**

| Paramètre              | Valeur                          | Rôle |
|------------------------|---------------------------------|------|
| `USE_REAL_IP`          | `yes`                           | Active la récupération de la vraie IP |
| `REAL_IP_HEADER`       | `CF-Connecting-IP`              | Header Cloudflare contenant l’IP réelle du client |
| `REAL_IP_FROM`         | Réseaux Docker + privés         | Adresses de confiance autorisées à fournir le header |
| `REAL_IP_RECURSIVE`    | `yes`                           | Recherche récursive dans le header |

### 5.3 Application de la configuration

```bash
cp /opt/bunkerweb/.env /opt/bunkerweb/.env.bak-$(date +%Y%m%d-%H%M)
# Édition du .env
cd /opt/bunkerweb
docker compose up -d
```

Vérification que les variables sont bien chargées :

```bash
docker inspect bunkerweb --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -i REAL_IP
```

Résultat obtenu :

```text
USE_REAL_IP=yes
REAL_IP_HEADER=CF-Connecting-IP
REAL_IP_FROM=172.16.0.0/12 172.28.0.0/16 172.29.0.0/16 10.0.0.0/8 192.168.0.0/16
REAL_IP_RECURSIVE=yes
```

![[Pasted image 20260819094534.png]]

---

## 6. Validation post-configuration

### 6.1 Test depuis un réseau externe (4G)

Après ouverture des sites depuis un téléphone en 4G, les logs montrent désormais la **vraie IP publique** du client :

```bash
docker logs --tail 50 bunkerweb | grep -E "client:|GET / HTTP"
```

Exemple de résultat :

```text
client: 44.197.147.75
horus-ais.com 44.197.147.75 - ... "GET / HTTP/1.1" 304 ...
```

![[Pasted image 20260819095207.png]]

L’IP interne `172.29.0.1` n’apparaît plus comme adresse client.

### 6.2 Comportement attendu désormais

- Les scanners et bots seront bannis sur leur **vraie adresse IP**
- L’IP du tunnel Cloudflare ne pourra plus être bannie par erreur
- Les logs d’accès deviennent exploitables pour l’analyse et les alertes

---

## 7. Points d’attention et améliorations futures

| Point | Statut | Commentaire |
|-------|--------|-------------|
| Transmission de la vraie IP | ✅ Résolu | Configuration Real IP active |
| Ban automatique du tunnel | ✅ Résolu | Ne peut plus se produire |
| Notifications de ban | ⏳ À faire | Intégration Wazuh + alertes email à venir |
| Affinage des logs BunkerWeb dans Wazuh | ⏳ En cours | Objectif principal de la session |
| Ajustement des seuils Bad Behavior | Optionnel | Possible si trop de faux positifs |

---

## 8. Checklist de clôture

- [x] Identification de la cause (ban de l’IP Docker `172.29.0.1`)
- [x] Levée du ban via `bwcli`
- [x] Sites de nouveau accessibles
- [x] Configuration `USE_REAL_IP` + `CF-Connecting-IP`
- [x] Vérification que les vraies IP publiques apparaissent dans les logs
- [x] Documentation de l’incident et de la correction

---

## 9. Références techniques

- Documentation BunkerWeb – Real IP : https://docs.bunkerweb.io/
- Headers Cloudflare : `CF-Connecting-IP`, `X-Forwarded-For`
- Plugin concerné : Bad Behavior
- Réseau Docker concerné : `bw-services` (`172.29.0.0/16`)

---

**Fin de la fiche 40**  
Document sanitizé pour publication (GitHub / portfolio).  
Version interne conservée séparément.
