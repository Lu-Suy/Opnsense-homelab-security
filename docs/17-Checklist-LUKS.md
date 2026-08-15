# 17 – Checklist : Réinstallation Debian 12 avec chiffrement LUKS

**Date originale :** 30 juillet 2026  
**Machine cible :** Prodesk (ancien bastion)  
**Version portfolio :** 15 août 2026 (sanitisée + enrichie)  
**Objectif :** Préparer une réinstallation propre avec chiffrement complet du système (LUKS) pour protéger les credentials et l’ensemble des données.

---

## Pourquoi une checklist de réinstallation chiffrée ?

Une réinstallation n’est jamais anodine. Sans préparation, on risque de :

- Perdre des configurations critiques (tunnel Cloudflare, volumes Docker, clés SSH…)
- Oublier des étapes de durcissement
- Se retrouver avec un système « qui marche à moitié »
- Sous-estimer le temps réel nécessaire

**Une checklist sert à :**

1. **Réduire le risque d’erreur humaine**  
   On ne compte pas sur la mémoire. Chaque étape est écrite et cochable.

2. **Protéger les secrets**  
   Les credentials Cloudflare (`/etc/cloudflared/*.json`) sont en clair sur le disque.  
   En cas d’accès physique au SSD (vol, maintenance, revente…), ils peuvent être extraits.  
   LUKS chiffre la partition entière → sans la passphrase, les données sont illisibles.

3. **Garantir la continuité de service**  
   Après réinstallation, le tunnel, BunkerWeb et les services doivent repartir correctement.  
   La checklist force à documenter ce qu’il faut sauvegarder et restaurer.

4. **Approche professionnelle**  
   Dans un contexte pro / data center, on ne « reformate et on verra ».  
   On prépare, on sauvegarde, on vérifie, on restaure, on valide.

C’est un outil de **maîtrise du changement**, pas seulement une liste de courses.

---

## 1. Contexte & motivation (rappel)

Actuellement les credentials du tunnel Cloudflare sont stockés **en clair** sur le disque.

Si quelqu’un obtient un accès physique au SSD ou un accès root, il peut récupérer ces secrets.

**Solution retenue :** Réinstaller Debian 12 avec **LUKS** (chiffrement de partition).

> **Note d’évolution :**  
> Cette checklist concernait la Prodesk (ancien bastion).  
> Le même principe a ensuite été appliqué sur le G4 (Rocky Linux + LUKS + déverrouillage distant via dracut-sshd – voir documents 23 et 27).

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
- Chiffrer 1 To en entier serait overkill et plus lent au démarrage / aux opérations I/O

---

## 3. Checklist de préparation (à faire avant la réinstallation)

### 3.1 Backup complet

- [ ] Brancher un disque externe (ou NAS)
- [ ] Faire un backup complet de `/` (rsync ou Clonezilla)
- [ ] Sauvegarder spécifiquement :
  - [ ] `/etc/cloudflared/` (config + credentials)
  - [ ] Volumes Docker de BunkerWeb
  - [ ] Répertoires home des utilisateurs d’administration
  - [ ] Configurations importantes du projet
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
- [ ] Recréer le(s) utilisateur(s) d’administration
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

- **Passphrase LUKS** : demandée à chaque démarrage (sauf unlock automatique via TPM ou déverrouillage distant – plus avancé).
- **Temps estimé** : 1h30 à 3h selon le rythme et la restauration.
- **Risque principal** : mauvaise restauration des volumes Docker → perte de config BunkerWeb.
- **Alternative temporaire** : garder le système actuel et renforcer uniquement les permissions + accès physique (moins robuste).

---

## 7. Décision pour la prochaine session

- [ ] Valider la taille exacte de la partition (100 Go ou 150 Go ?)
- [ ] Choisir la méthode de backup (rsync vs Clonezilla)
- [ ] Préparer le disque externe
- [ ] Planifier le créneau (idéalement quand on a plusieurs heures devant soi)

---

**Document créé le 30 juillet 2026**  
**À utiliser comme checklist pour une réinstallation chiffrée maîtrisée.**

**Document de référence – Checklist réinstallation Debian + LUKS**  
*Version portfolio sanitisée et enrichie – 15 août 2026*
