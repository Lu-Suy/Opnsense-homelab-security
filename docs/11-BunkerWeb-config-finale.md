# 11 – BunkerWeb configuration finale + virtual hosts + HTTPS Let’s Encrypt

**Date originale** : 21 mai 2026  
**Objectif** : Passer en multisite avec certificats Let’s Encrypt automatiques  
**Version BunkerWeb** : 1.6.10 (mise à jour effectuée le 21/05/2026)  
**Version portfolio** : 15 août 2026 (enrichie + sanitisée)

> **Note portfolio** :  
> Snapshot historique de la première tentative de multisite (mai 2026 – ancien bastion).  
> Les domaines et IPs ont été généralisés.  
> Contenu technique conservé intégralement.  
> Cette approche a ensuite évolué vers le multisite actuel (docs 21 et 26) + Cloudflare Tunnel.

---

## Philosophie : pourquoi passer en multisite + Let’s Encrypt ?

À ce stade du projet, BunkerWeb servait déjà une page statique sur l’IP services.  
L’objectif suivant était de :

1. **Gérer plusieurs domaines** sur la même instance (multisite)
2. **Obtenir des certificats HTTPS gratuits et automatiques** via Let’s Encrypt
3. Préparer l’exposition publique des sites de façon propre

**Pourquoi Let’s Encrypt plutôt qu’un certificat auto-signé ?**
- Les navigateurs font confiance aux certificats Let’s Encrypt
- Renouvellement automatique
- Standard de l’industrie pour les sites publics

**Pourquoi le challenge HTTP-01 à cette époque ?**
- Plus simple à mettre en place que le challenge DNS-01
- BunkerWeb gère automatiquement les fichiers de challenge sur le port 80
- Adapté tant que les domaines pointent correctement vers l’IP publique

---

## 1. État avant multisite

- BunkerWeb 1.6.9 → mis à jour vers 1.6.10  
- Page statique fonctionnelle sur `http://10.0.10.x`  
- Dossier : `/opt/bunkerweb`

---

## 2. Configuration multisite (à faire maintenant)

### 2.1 Édition du fichier `.env` (le plus simple et propre)

```bash
cd /opt/bunkerweb
nano .env
```

Ajoute ou modifie ces lignes à la fin du fichier (adapte avec tes domaines) :

```env
# === MULTISITE ===
MULTISITE=yes
SERVER_NAME=exemple1.domain exemple2.domain exemple3.domain

# === Domaine 1 ===
exemple1.domain_USE_LETS_ENCRYPT=yes
exemple1.domain_LETS_ENCRYPT_EMAIL=ton-email@exemple.com
exemple1.domain_LETS_ENCRYPT_DNS_CHALLENGE=http   # HTTP-01 (le plus simple)

# === Domaine 2 ===
exemple2.domain_USE_LETS_ENCRYPT=yes
exemple2.domain_LETS_ENCRYPT_EMAIL=ton-email@exemple.com
exemple2.domain_LETS_ENCRYPT_DNS_CHALLENGE=http

# === Domaine 3 ===
exemple3.domain_USE_LETS_ENCRYPT=yes
exemple3.domain_LETS_ENCRYPT_EMAIL=ton-email@exemple.com
exemple3.domain_LETS_ENCRYPT_DNS_CHALLENGE=http

# (ajoute les autres domaines de la même façon)
```

**Effet** :  
- `MULTISITE=yes` active le mode multisite.  
- `SERVER_NAME` liste tous les domaines.  
- `USE_LETS_ENCRYPT=yes` + `DNS_CHALLENGE=http` active le challenge HTTP-01 (BunkerWeb va créer automatiquement les challenges sur le port 80).  
- Pas besoin de DNS-01 pour l’instant (plus simple).

Sauvegarde (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 2.2 Redémarrage propre

```bash
docker compose down
docker compose up -d --force-recreate
docker compose ps
```

**Effet** : BunkerWeb recharge la nouvelle configuration multisite.

---

## 3. Configuration DNS (à faire en parallèle)

Dans le dashboard du registrar / Unstoppable Domains (ou équivalent) :
- Pour chaque domaine :
  - Crée un enregistrement **A** pointant vers ton **IP publique WAN** (celle visible depuis Internet).
  - Optionnel : un enregistrement **CNAME** `www` → le domaine principal.

---

## 4. Vérification finale

- Accède à `http://exemple1.domain` (une fois le DNS propagé).  
- BunkerWeb doit rediriger automatiquement vers HTTPS et demander le certificat Let’s Encrypt.

**Prochaines étapes après cette config** :
- Vérification des certificats (`docker logs bunkerweb --tail 50`).
- Ajout des pages personnalisées par domaine dans le dossier `www`.
- Durcissement supplémentaire (CrowdSec, etc.).

---

**Dernière mise à jour originale** : 21 mai 2026 – BunkerWeb 1.6.10 + multisite activé.

---

**Note d’évolution (août 2026)** :  
Cette configuration correspond à la première phase multisite sur l’ancien bastion.  
L’architecture actuelle utilise Cloudflare Tunnel + BunkerWeb multisite sur le G4 (voir documents 21 et 26), ce qui change la façon dont les certificats et l’exposition publique sont gérés.

---

**Document historique – Première configuration multisite BunkerWeb + Let’s Encrypt**  
*Version portfolio enrichie – 15 août 2026*
