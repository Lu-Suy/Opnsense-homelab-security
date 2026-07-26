
**Date** : 21 mai 2026  
**Objectif** : Passer en multisite avec tous les domaines Unstoppable + certificats Let’s Encrypt automatiques.

**Version actuelle** : BunkerWeb 1.6.10 (mise à jour effectuée le 21/05/2026).

## 1. État avant multisite
- BunkerWeb 1.6.9 → mis à jour vers 1.6.10  
- Page statique fonctionnelle sur `http://10.0.10.10`  
- Dossier : `/opt/bunkerweb`

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
SERVER_NAME=godmode.her godmode.privacy iagodmode.her iagodmode.privacy iasex.her

# === Domaine 1 : godmode.her ===
godmode.her_USE_LETS_ENCRYPT=yes
godmode.her_LETS_ENCRYPT_EMAIL=ton-email@exemple.com
godmode.her_LETS_ENCRYPT_DNS_CHALLENGE=http   # HTTP-01 (le plus simple pour Unstoppable)

# === Domaine 2 : godmode.privacy ===
godmode.privacy_USE_LETS_ENCRYPT=yes
godmode.privacy_LETS_ENCRYPT_EMAIL=ton-email@exemple.com
godmode.privacy_LETS_ENCRYPT_DNS_CHALLENGE=http

# === Domaine 3 : iagodmode.her ===
iagodmode.her_USE_LETS_ENCRYPT=yes
iagodmode.her_LETS_ENCRYPT_EMAIL=ton-email@exemple.com
iagodmode.her_LETS_ENCRYPT_DNS_CHALLENGE=http

# (ajoute les autres domaines de la même façon : iagodmode.privacy, iasex.her, etc.)
```

**Effet** :  
- `MULTISITE=yes` active le mode multisite.  
- `SERVER_NAME` liste tous tes domaines.  
- `USE_LETS_ENCRYPT=yes` + `DNS_CHALLENGE=http` active le challenge HTTP-01 (BunkerWeb va créer automatiquement les challenges sur port 80).  
- Pas besoin de DNS-01 pour l’instant (plus simple avec tes domaines Unstoppable).

Sauvegarde (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 2.2 Redémarrage propre

```bash
docker compose down
docker compose up -d --force-recreate
docker compose ps
```

**Effet** : BunkerWeb recharge la nouvelle configuration multisite.

## 3. Configuration DNS sur Unstoppable Domains (à faire en parallèle)

Dans ton dashboard Unstoppable :
- Pour chaque domaine (`godmode.her`, etc.) :
  - Crée un enregistrement **A** pointant vers ton **IP WAN publique** (celle de ton OPNsense : 192.168.5.244 ou l’IP publique visible sur whatismyip.com).
  - Optionnel : un enregistrement **CNAME** `www` → le domaine principal.

## 4. Vérification finale

- Accède à `http://godmode.her` (une fois le DNS propagé).  
- BunkerWeb doit rediriger automatiquement vers HTTPS et demander le certificat Let’s Encrypt.

**Prochaines étapes après cette config** :
- Vérification des certificats (`docker logs bunkerweb --tail 50`).
- Ajout des pages personnalisées par domaine dans le dossier `www`.
- Durcissement supplémentaire (CrowdSec, etc.).

**Dernière mise à jour** : 21 mai 2026 – BunkerWeb 1.6.10 + multisite activé.

---
