# 37 – Maintenance G4 du 17 août 2026  
**Upgrade système + Correctif multi-LUKS + BunkerWeb 1.6.13**

**Projet :** Bastion Godmode / Horus AIS  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Date :** 17 août 2026  
**Auteur :** Ludovic + Grok (équipe)  

---

## 1. Objectif

Réaliser une maintenance système complète sur le bastion G4 :
- Mise à jour de tous les paquets (kernel, dracut, Docker, correctifs sécurité)
- Passage de BunkerWeb de 1.6.10 à 1.6.13
- Correction définitive d’un problème de boot multi-LUKS découvert après le reboot

---

## 2. État avant maintenance

| Composant              | Version / État                          |
|------------------------|-----------------------------------------|
| Kernel running         | 6.12.0-211.42.1.el10_2                 |
| Wazuh                  | 4.14.7-1 (Indexer + Manager + Dashboard)|
| BunkerWeb              | 1.6.10                                  |
| Volume de backup LUKS  | Configuré dans `/etc/crypttab` **sans** `noauto` |
| Services critiques     | Opérationnels                           |

---

## 3. Actions réalisées

### 3.1 Mise à jour système complète

```bash
sudo dnf check-update
sudo dnf update -y
```

**Principaux paquets mis à jour :**
- Kernel → **6.12.0-211.47.1.el10_2**
- dracut (107-9)
- Docker CE + containerd + docker-compose-plugin
- curl / libcurl, libgcrypt, libnghttp2
- SELinux policy, NetworkManager, firmwares, etc.

### 3.2 Upgrade BunkerWeb 1.6.10 → 1.6.13

Le fichier `docker-compose.yml` pinait la version en dur.  
Modification effectuée :

```yaml
# Avant
image: bunkerity/bunkerweb:1.6.10
image: bunkerity/bunkerweb-scheduler:1.6.10

# Après
image: bunkerity/bunkerweb:1.6.13
image: bunkerity/bunkerweb-scheduler:1.6.13
```

Puis :

```bash
cd /opt/bunkerweb
sudo docker compose pull
sudo docker compose up -d
```

**Résultat final :**
```
bunkerweb          bunkerity/bunkerweb:1.6.13
bw-scheduler       bunkerity/bunkerweb-scheduler:1.6.13
```

---
BunkerWeb a été mis à jour vers la version **1.6.13** (dernière stable au 17 août 2026).

```
bunkerweb          bunkerity/bunkerweb:1.6.13
bw-scheduler       bunkerity/bunkerweb-scheduler:1.6.13
```

#### Gestion des versions d’images Docker

Trois approches sont possibles pour gérer la version des images :

| Méthode | Exemple | Avantages | Inconvénients | Recommandé pour un bastion |
|---------|---------|-----------|---------------|----------------------------|
| **Version hardcodée** | `1.6.13` | Très contrôlé, reproductible | Nécessite de modifier le `docker-compose.yml` à chaque upgrade | Oui (le plus sûr) |
| **Variable dans `.env`** | `BUNKERWEB_VERSION=1.6.13` | Plus propre, un seul endroit à modifier | Toujours manuel | **Oui – meilleure pratique** |
| **`latest` ou `1.6`** | `bunkerity/bunkerweb:latest` | Toujours la dernière version | Risque de casser en production (mise à jour non contrôlée) | **Non** |

**Recommandation retenue :**
- Ne pas utiliser le tag `latest`.
- Conserver un contrôle strict des versions (approche bastion / production).
- Amélioration possible : déplacer la version dans le fichier `.env` pour simplifier les prochains upgrades.

Exemple de bonne pratique :

```env
BUNKERWEB_VERSION=1.6.13
```

Dans le `docker-compose.yml` :

```yaml
image: bunkerity/bunkerweb:${BUNKERWEB_VERSION}
image: bunkerity/bunkerweb-scheduler:${BUNKERWEB_VERSION}
```


---

## 4. Incident rencontré – Multi-LUKS au reboot

### Description

Après le `dnf update` + reboot :

1. Déverrouillage distant du volume **système** via dracut-sshd → réussi
2. La session SSH de dracut se ferme (comportement normal)
3. Le système continue le boot et demande **interactivement** la passphrase du volume de **backup** (`/dev/sda` → `backup_crypt`)
4. Tant que cette passphrase n’est pas fournie, le boot n’est pas terminé → le service SSH normal n’est pas accessible

**Cause racine :**  
Dans la fiche 35, la ligne suivante avait été ajoutée dans `/etc/crypttab` :

```text
backup_crypt UUID=2b5e7368-42af-4e7a-a4d7-749409974f0a none luks
```

Sans les options `noauto` et `nofail`.  
Après l’update de dracut + nouveau kernel, le comportement systemd est devenu bloquant.

### Contournement immédiat

Accès physique (clavier + écran) pour saisir manuellement la passphrase du volume de backup.

---

## 5. Correctif appliqué

### 5.1 Modification de `/etc/crypttab`

**Avant :**
```text
backup_crypt UUID=2b5e7368-42af-4e7a-a4d7-749409974f0a none luks
```

**Après :**
```text
backup_crypt UUID=2b5e7368-42af-4e7a-a4d7-749409974f0a none luks,noauto,nofail
```

### 5.2 Commandes

```bash
sudo nano /etc/crypttab
sudo systemctl daemon-reload
```

**Effet des options :**
- `noauto` → le volume n’est **jamais** déverrouillé automatiquement au boot
- `nofail` → même en cas d’échec sur ce volume, le boot continue

Le volume de backup doit désormais être ouvert et monté **manuellement** quand on en a besoin :

```bash
sudo cryptsetup open /dev/sda1 backup_crypt
sudo mount /mnt/backup_g4
```

---

## 6. État final après maintenance

| Élément                    | Valeur / Statut                          |
|----------------------------|------------------------------------------|
| Kernel                     | **6.12.0-211.47.1.el10_2**              |
| Wazuh                      | 4.14.7-1 (tous composants active)       |
| BunkerWeb                  | **1.6.13** (healthy)                    |
| Docker                     | Actif                                   |
| `/etc/crypttab`            | Corrigé avec `noauto,nofail`            |
| Volume de backup           | Non monté automatiquement (comportement voulu) |
| Services critiques         | Tous `active`                           |

---

## 7. Leçons apprises & recommandations

1. **Toujours** ajouter `noauto,nofail` (ou au minimum `noauto`) sur les volumes LUKS secondaires qui ne sont pas le root.
2. Après toute mise à jour de **dracut** ou du kernel, tester un reboot avec un accès console disponible.
3. Prévoir un accès out-of-band (IPMI / iLO / KVM) pour les prochains équipements si possible.
4. Documenter immédiatement les incidents de boot (cette fiche en est l’exemple).
5. Pour BunkerWeb : envisager de déplacer la version dans le fichier `.env` afin de simplifier les futurs upgrades.

---

## 8. Liens avec les fiches existantes

- **Fiche 27** – Déverrouillage distant LUKS via dracut-sshd  
  → Ajouter une note : « Voir fiche 37 pour le correctif multi-LUKS après update dracut/kernel »

- **Fiche 35** – Volume de backup interne chiffré (LUKS)  
  → Confirmer que la ligne crypttab contient désormais `noauto,nofail`

---

## 9. Checklist de validation post-maintenance

- [x] Nouveau kernel actif
- [x] Services Wazuh actifs
- [x] BunkerWeb 1.6.13 healthy
- [x] `/etc/crypttab` corrigé
- [x] Boot ne demande plus la passphrase du volume de backup
- [ ] Volume de backup monté manuellement si besoin
- [ ] Sites web accessibles via Cloudflare Tunnel

---

**Document rédigé le 17 août 2026 (soir)**  

