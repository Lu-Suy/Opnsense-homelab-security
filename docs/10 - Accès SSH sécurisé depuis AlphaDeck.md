
**Date** : 08 juillet 2026
**Objectif** : Connexion SSH depuis AlphaDeck vers OPNsense (LAN)

## Configuration actuelle

- **Utilisateur** : `d**` (créé via GUI System → Access → Users)
**1. Group membership + Shell pour doo**

- **System → Access → Users → édite d****
    - **Group Membership** : ajoute-le au groupe **wheel** ou **admins** (c’est ce qui donne les droits sudo).
    - **Login Shell** : mets **/bin/sh** (très important, sinon SSH refuse souvent).
    - **Authorized keys** : la clé doit être collée (vérifie).
     
- **Save**

**2. Applique les changements** Va dans **System → Settings → Administration** → **Apply Changes** (même si rien n’a changé, ça recharge SSH).


![Capture d’écran](../images/Pasted%20image%2020260708192552.png)

id_ed25519.pub : 

- **Méthode** : Clé SSH + passphrase + password fallback
- **Clé publique** : ed25519 (clé privée protégée par passphrase)
- **Interface** : LAN (IGC2)
- **Port** : 22
- **Secure Shell GUI** :
  - Enable Secure Shell : coché
  - Permit root user login : coché
  - Permit password login : décoché (mais fallback encore actif)


![Capture d’écran](../images/Pasted%20image%2020260708192926.png)


## Test de connexion depuis AlphaDeck

```powershell
ssh d**@10.0.0.*
```

- Demande la passphrase de la clé privée
- Puis demande le mot de passe utilisateur `d**` en fallback

## Remarques

- La connexion par clé SSH fonctionne.
- On garde la passphrase et le password fallback pour l’instant.
- Durcissement complet (clé uniquement sans passphrase ni password) sera fait plus tard dans le fichier Hardening.

Si besoin : Corrige les permissions Windows

PowerShell

```
# Donne les bons droits (très important sur Windows)
icacls $HOME\.ssh\id_ed25519 /inheritance:r /grant:r $env:USERNAME`:F
icacls $HOME\.ssh /inheritance:r /grant:r $env:USERNAME`:F
```

**Version encore plus sûre (à exécuter en PowerShell Admin)** :

PowerShell

```
cd $HOME\.ssh
icacls id_ed25519 /inheritance:r /grant:r $env:USERNAME`:F
icacls id_ed25519.pub /inheritance:r /grant:r $env:USERNAME`:F
icacls . /inheritance:r /grant:r $env:USERNAME`:F
```

### Solution sans renommer (avec -i)

**Commande complète à utiliser à chaque fois** (ou tu peux créer un alias plus tard) :

PowerShell

```
ssh -i $HOME\.ssh\id_ed25519 root@10.0.0.*
```

**Version verbose pour debug** :

PowerShell

```
ssh -v -i $HOME\.ssh\id_ed25519 root@10.0.0.*
```

**Pour l’utilisateur doo** :

PowerShell

```
ssh -v -i $HOME\.ssh\id_ed25519 doo@10.0.0.*
```

## Prochaines étapes (Hardening)

- Désactiver complètement le login par mot de passe
- Changer le port SSH ou utiliser un port non standard
- Mettre en place fail2ban / 2FA si nécessaire
- Créer un utilisateur dédié sans privilèges root pour l’administration courante


**Dernière mise à jour** : 08 juillet 2026.

