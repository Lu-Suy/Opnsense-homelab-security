# 17 – Checklist : Réinstallation Debian 12 avec chiffrement LUKS (Prodesk)

**Projet :** Bastion Godmode  
**Date de création :** 30 juillet 2026  
**Objectif :** Préparer une réinstallation propre de la Prodesk avec chiffrement complet du système (LUKS) pour protéger les credentials Cloudflare Tunnel et l’ensemble des données.

---

## 1. Contexte & motivation

Actuellement les credentials du tunnel Cloudflare (`/etc/cloudflared/*.json`) sont stockés **en clair** sur le disque.

Si quelqu’un obtient un accès physique au SSD ou un accès root, il peut récupérer ces secrets.

**Solution retenue :** Réinstaller Debian 12 avec **LUKS** (chiffrement de partition).

---

## 2. Dimensionnement de la partition

| Élément                        | Recommandation          | Commentaire |
|--------------------------------|-------------------------|-------------|
| Taille du SSD                  | 1 To                    | — |
| Partition système chiffrée     | **100 – 150 Go**        | Largement suffisant |
| Partition data (optionnelle)   | Reste du disque         | Non chiffrée ou chiffrée séparément |
| Swap                           | 4 – 8 Go (chiffré)      | Recommandé |

**Pourquoi 100-150 Go suffisent ?**
- Debian 12 de base + Docker + BunkerWeb + cloudflared
- Images Docker + volumes
- Logs et marge de manœuvre
- 1 To chiffré en entier serait overkill et plus lent

---

## 3. Checklist de préparation (à faire avant la réinstallation)

### 3.1 Backup complet

- [ ] Brancher un disque externe (ou NAS)
- [ ] Faire un backup complet de `/` (rsync ou Clonezilla)
- [ ] Sauvegarder spécifiquement :
  - [ ] `/etc/cloudflared/` (config + credentials)
  - [ ] Volumes Docker de BunkerWeb
  - [ ] `/home/hyper_doo/`
  - [ ] Configurations OPNsense liées (si besoin)
  - [ ] Fichiers de documentation du projet
- [ ] Vérifier l’intégrité du backup (test de restauration d’un fichier)

### 3.2 Inventaire à documenter

- [ ] Liste des conteneurs Docker (`docker ps -a`)
- [ ] Liste des volumes Docker (`docker volume ls`)
- [ ] Contenu de `/etc/cloudflared/`
- [ ] Utilisateurs et groupes importants
- [ ] Clés SSH autorisées
- [ ] Configuration réseau actuelle (IPs, etc.)

---

## 4. Pendant l’installation Debian

- [ ] Choisir le mode d’installation **Guided – use entire disk and set up encrypted LVM** (ou manuel)
- [ ] Créer une partition root chiffrée de **100-150 Go**
- [ ] Activer le chiffrement LUKS
- [ ] Noter la passphrase LUKS dans un endroit sûr (gestionnaire de mots de passe)
- [ ] Installer uniquement le système de base + SSH server
- [ ] Ne pas cocher les bureaux graphiques (machine headless)

---

## 5. Après la réinstallation

### 5.1 Remise en service de base

- [ ] Mettre à jour le système (`apt update && apt full-upgrade`)
- [ ] Recréer l’utilisateur `hyper_doo`
- [ ] Restaurer les clés SSH
- [ ] Installer Docker + Docker Compose
- [ ] Restaurer les volumes / configs BunkerWeb
- [ ] Réinstaller `cloudflared`
- [ ] Restaurer `/etc/cloudflared/`
- [ ] Relancer le service `cloudflared`
- [ ] Vérifier que `https://horus-ais.com` répond

### 5.2 Durcissement post-install

- [ ] Permissions strictes sur `/etc/cloudflared/`
- [ ] Fail2ban
- [ ] Configuration SSH durcie
- [ ] Mise à jour du fichier d’inventaire hardware (14)

---

## 6. Points d’attention

- **Passphrase LUKS** : à chaque démarrage (sauf unlock automatique via TPM – plus complexe et moins sécurisé).
- **Temps estimé** : 1h30 à 3h selon le rythme et la restauration.
- **Risque** : mauvaise restauration des volumes Docker → perte de config BunkerWeb.
- **Alternative temporaire** : garder le système actuel et renforcer uniquement les permissions + accès physique.

---

## 7. Décision pour la prochaine session

- [ ] Valider la taille exacte de la partition (100 Go ou 150 Go ?)
- [ ] Choisir la méthode de backup (rsync vs Clonezilla)
- [ ] Préparer le disque externe
- [ ] Planifier le créneau (idéalement quand on a plusieurs heures devant soi)

---

**Document créé le 30 juillet 2026**  
**À utiliser comme checklist pour la session de réinstallation chiffrée de la Prodesk.**
