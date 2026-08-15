# 10 – Accès SSH sécurisé depuis la workstation d’administration

**Date originale** : 08 juillet 2026  
**Objectif** : Connexion SSH depuis la workstation d’administration vers OPNsense (LAN)  
**Version portfolio** : 15 août 2026 (enrichie + sanitisée)

> **Note portfolio** :  
> Document enrichi pour expliquer le *pourquoi* des choix.  
> Les usernames, IPs et hostnames ont été généralisés.  
> **Toutes les captures d’écran et le contenu technique original sont conservés.**

---

## Philosophie : pourquoi sécuriser l’accès SSH dès le début ?

L’accès SSH à OPNsense est l’un des points les plus critiques de l’infrastructure.

**Pourquoi le traiter avec attention ?**
- OPNsense est le **point central** de sécurité (firewall, routing, IDS…). Une compromission ici = compromission de toute l’architecture.
- SSH est un protocole puissant : une fois authentifié, on a un accès shell complet.
- Les attaques par force brute sur le port 22 sont constantes sur Internet. Même sur un réseau local, on applique les bonnes pratiques.

**Objectifs de cette configuration :**
1. Authentification par **clé SSH** (plus robuste qu’un mot de passe seul)
2. Restriction des utilisateurs autorisés
3. Préparation au durcissement complet (désactivation du password login, etc.)

---

## Configuration actuelle

- **Utilisateur** : `limited-user` (créé via GUI System → Access → Users)

### 1. Group membership + Shell pour limited-user

- **System → Access → Users → édite limited-user**
    - **Group Membership** : ajoute-le au groupe **wheel** ou **admins** (c’est ce qui donne les droits sudo).
    - **Login Shell** : mets **/bin/sh** (très important, sinon SSH refuse souvent).
    - **Authorized keys** : la clé doit être collée (vérifie).
     
- **Save**

### 2. Applique les changements

Va dans **System → Settings → Administration** → **Apply Changes** (même si rien n’a changé, ça recharge SSH).

![Capture d’écran](../images/Pasted%20image%2020260708192552.png)

**Clé publique** : ed25519 (clé privée protégée par passphrase)

- **Méthode** : Clé SSH + passphrase + password fallback
- **Interface** : LAN
- **Port** : 22
- **Secure Shell GUI** :
  - Enable Secure Shell : coché
  - Permit root user login : coché (à durcir plus tard)
  - Permit password login : décoché (mais fallback encore actif)

![Capture d’écran](../images/Pasted%20image%2020260708192926.png)

### Pourquoi ces choix à ce stade ?

- **Clé ed25519 + passphrase** : meilleur compromis sécurité / praticité. La passphrase protège la clé privée même si le fichier est copié.
- **Password fallback encore actif** : permet de ne pas se retrouver bloqué pendant la phase de mise en place.
- **Permit root login encore coché** : pratique en phase d’installation, à désactiver ensuite (voir documents de hardening).

---

## Test de connexion depuis la workstation

```powershell
ssh limited-user@10.0.0.z
```

- Demande la passphrase de la clé privée
- Puis demande le mot de passe utilisateur `limited-user` en fallback

---

## Remarques

- La connexion par clé SSH fonctionne.
- On garde la passphrase et le password fallback pour l’instant.
- Durcissement complet (clé uniquement sans password) sera fait plus tard dans les documents de Hardening.

### Si besoin : Corriger les permissions Windows

Sur Windows, les permissions des fichiers de clé SSH sont critiques. Si elles sont trop ouvertes, SSH refuse d’utiliser la clé.

**PowerShell :**

```powershell
# Donne les bons droits (très important sur Windows)
icacls $HOME\.ssh\id_ed25519 /inheritance:r /grant:r $env:USERNAME`:F
icacls $HOME\.ssh /inheritance:r /grant:r $env:USERNAME`:F
```

**Version encore plus sûre (à exécuter en PowerShell Admin)** :

```powershell
cd $HOME\.ssh
icacls id_ed25519 /inheritance:r /grant:r $env:USERNAME`:F
icacls id_ed25519.pub /inheritance:r /grant:r $env:USERNAME`:F
icacls . /inheritance:r /grant:r $env:USERNAME`:F
```

### Solution sans renommer (avec -i)

**Commande complète à utiliser à chaque fois** (ou tu peux créer un alias plus tard) :

```powershell
ssh -i $HOME\.ssh\id_ed25519 limited-user@10.0.0.z
```

**Version verbose pour debug** :

```powershell
ssh -v -i $HOME\.ssh\id_ed25519 limited-user@10.0.0.z
```

---
### Rester sur le port 22 – est-ce une erreur ?

**Non, ce n’est pas une erreur**, mais c’est un **choix conscient** avec des avantages et des inconvénients.

| Approche                               | Avantages                                                               | Inconvénients                                                                                                |
| -------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Garder le port 22**                  | Standard, simple, compatible avec tous les outils, documentation claire | Cible privilégiée des scanners et bots de brute-force                                                        |
| **Changer de port** (ex. 2222, 22022…) | Réduit fortement le bruit des scans automatiques                        | Security through obscurity (pas une vraie sécurité), il faut se souvenir du port, plus de config à maintenir |

Nous avons volontairement conservé le port SSH standard (22).

Bien que changer de port puisse réduire le bruit des scanners automatiques, cela relève principalement de la « security through obscurity » et n’apporte qu’une protection limitée.

Dans notre architecture, l’accès SSH est déjà fortement restreint par les règles firewall (uniquement depuis la workstation d’administration), renforcé par Fail2ban et l’authentification par clé. De plus, le port 22 n’est pas exposé directement sur Internet.

Dans ces conditions, conserver le port standard offre un meilleur compromis entre sécurité réelle, clarté opérationnelle et maintenabilité.


## Prochaines étapes (Hardening)

- Désactiver complètement le login par mot de passe
- Changer le port SSH ou utiliser un port non standard (optionnel)
- Mettre en place fail2ban / 2FA si nécessaire
- Créer un utilisateur dédié sans privilèges root pour l’administration courante

Ces points sont traités dans les documents :
- [07b – Hardening du Bastion](./07b%20-%20Hardening%20du%20Bastion%20Godmode.md)
- [28 – Durcissement SSH (G4)](./28%20-%20Durcissement%20SSH%20(G4).md)
- [29 – Fail2ban protection brute-force SSH (G4)](./29%20-%20Fail2ban%20protection%20brute-force%20SSH%20(G4).md)

---

**Dernière mise à jour originale** : 08 juillet 2026.

**Document de référence – Accès SSH sécurisé vers OPNsense**  
*Version portfolio enrichie – 15 août 2026*
