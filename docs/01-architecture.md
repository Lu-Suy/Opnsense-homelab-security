# 01 - Architecture & Sommaire du Vault Obsidian - Bastion Godmode

**Projet** : Bastion Godmode  
**Date de création** : 15 mai 2026  
**Dernière mise à jour** : 18 mai 2026  
**Objectif du vault** : Documenter l’ensemble de l’infrastructure homelab de façon structurée, reproductible, sécurisée et versionnée.

## Topologie actuelle du réseau (18 mai 2026)

- **OPNsense** : HP EliteDesk 800 G2 SFF  
  - IGC3 → WAN (192.168.5.244/24)  
  - IGC2 → LAN (10.0.0.1/24)  
  - IGC1 → **OPT1** (10.0.10.1/24) ← Réseau principal du Bastion  
  - IGC0 → OPT2 (10.0.20.1/24)  

- **Bastion principal (Prodesk 600 Mini G2)** :  
  - Debian 12 fraîche sur SSD 1 To  
  - IP Web/Service : **10.0.10.10/24** (BunkerWeb)  
  - IP Manager/Admin : **10.0.10.11/24** (SSH durci)  

- **Machines clientes** : AlphaDeck (10.0.0.10 sur LAN), laptop, téléphone, etc.  
- **Suricata** : Mode IDS passif sur toutes les interfaces OPNsense.

![Schéma réseau local](../images/Schéma_Réseau_local.png)

## Stack technique actuel (18 mai 2026)

**En place et fonctionnel** :
- Firewall : **OPNsense 26.1.8_5** (à jour, CVE patchées)
- WAF / Reverse-Proxy : **BunkerWeb 1.6.9** (Docker) → Page statique personnalisée avec cœurs roses fonctionnelle sur `http://10.0.10.10`
- Accès administrateur : SSH clé ED25519 uniquement + Fail2ban actif
- Séparation privilèges : `doo` (utilisateur limité) + `hyper_doo` (admin full sudo)
- Monitoring : NetFlow + Insight configuré sur OPNsense
- IDS : Suricata (passif)

**En cours / partiellement fait** :
- Hardening global Prodesk
- Documentation Obsidian (structure propre)

**Objectif final** :
- OpenVPN / WireGuard (accès distant sécurisé)
- Wazuh SIEM complet (manager + agents + corrélation Suricata)
- BunkerWeb multisite + Let’s Encrypt (domaines .com & web3, etc.)
- Monitoring avancé + alerting
- Disk encryption + backup automatisés

## Structure générale du vault
```
Bastion-Godmode/
├── docs/                  ← Tous les documents techniques numérotés
├── images/                ← Captures d’écran et schémas
├── Contexte à incrémenter ← Notes rapides / idées futures
├── README.md
└── recap projet Bastion Godmode au 14 mai 2026.
```

## Liste des fichiers docs/ (numérotation chronologique – 04/07/2026)

| N°  | Nom du fichier                                            | Statut     | Contenu principal                              |
| --- | --------------------------------------------------------- | ---------- | ---------------------------------------------- |
| 00  | Backup & réinstallation Prodesk                           | Terminé    | Procédure complète de backup et réinstallation |
| 01  | Architecture & Sommaire du Vault Obsidian                 | À jour     | Ce fichier (sommaire global)                   |
| 02  | Network                                                   | À jour     | Schéma réseau + interfaces OPNsense            |
| 03  | État actuel du Firewall                                   | En cours   | Règles globales actuelles                      |
| 04  | Firewall Rules Hardening & Cleanup                        | En cours   | Version détaillée avec logs et tableaux        |
| 04b | 04b - VLANs et Segmentation Réseau                        | En cours   | VLANs                                          |
| 05  | État de la machine Prodesk (Bastion Godmode)              | Terminé    | Hardware, software, état actuel                |
| 06  | Aliases OPNsense                                          | Terminé    | Liste complète des aliases                     |
| 07  | Hardening du Bastion Godmode                              | Terminé    | Hardening global                               |
| 07b | Hardening du Bastion Godmode                              | Terminé    | Version détaillée                              |
| 07C | Pentest Externe Contrôlé                                  | En cours   | Tests d’attaque contrôlés                      |
| 08  | BunkerWeb Install                                         | Terminé    | Installation et config de base                 |
| 09  | Suricata (Intrusion Detection) sur OPNsense               | En cours   | Configuration IDS                              |
| 10  | Accès SSH sécurisé depuis AlphaDeck                       | Terminé    | Configuration SSH durcie                       |
| 11  | BunkerWeb configuration finale + virtual hosts            | À faire    | Configuration multisite + HTTPS                |
| 11b | Configuration Port Forwarding sur Nordnet                 | En cours   | Règles NAT Nordnet                             |
| 12  | Domaine & Messagerie externe (horus-ais.com + OVH Zimbra) | ✅ Terminé  |                                                |
| 12a | Mise en service de la messagerie OVH Zimbra               | ✅ Terminé  | SPF/DKIM/DMARC OK                              |
| 13  | Installation Zsh + Starship + Grok Build (Prodesk)        | ✅ Terminé  |                                                |
| 13a | Version détaillée Zsh/Starship/Grok Build                 | ✅ Terminé  |                                                |
| 14  | Inventaire Hardware & Ressources des machines             | ✅ Terminé  |                                                |
| 15  | Objectif du Cloudflare Tunnel                             | ✅ Terminé  |                                                |
| 16  | Installer cloudflared sur Debian                          | ✅ Terminé  |                                                |
| 16a | Installer cloudflared et Cloudflare Tunnel (Prodesk)      | ✅ Terminé  | Version synthétique                            |
| 17  | Checklist réinstallation Debian avec LUKS                 | ⏳ Prêt     | Pas encore exécuté                             |
| 18  | Est-ce que BunkerWeb protège vraiment ta page index       | ✅ Terminé  | Analyse / tests                                |
| 18A | Alignement projet Bastion Godmode (30 juillet 2026)       | ✅ Terminé  | Point de référence                             |
| 19  | Migration des règles Firewall OPNsense vers MVC_API       | ✅ Terminé  |                                                |
| 19A | Optimisation STUN/TURN + alias                            | ✅ Terminé  |                                                |
| 20  | Hardening cloudflared – utilisateur dédié non-root        | ✅ Terminé  |                                                |
| 20a | Version synthétique hardening cloudflared                 | ✅ Terminé  |                                                |
| 21  | Multisite BunkerWeb                                       | ✅ Terminé  |                                                |
| 21a | Version synthétique Multisite BunkerWeb                   | ✅ Terminé  |                                                |
| 21b | Redirection mrdoolux.brave (Unstoppable)                  | ⏳ En cours | Pending validation Unstoppable                 |

12Domaine & Messagerie externe (horus-ais.com + OVH Zimbra)✅ Terminé
## Fichiers à créer prochainement (ordre logique recommandé)

|       |                                                        |                                                          |
| ----- | ------------------------------------------------------ | -------------------------------------------------------- |
| Ordre | Fichier proposé                                        | Objectif                                                 |
| 22    | 22 - Redirection Unstoppable mrdoolux.brave (final).md | Clôturer proprement la redirection Web3 une fois validée |
| 23    | 23 - Contenu site horus-ais.com & mrdoolux.md          | Structure + premières pages réelles                      |
| 24    | 24 - Mise en service GX10 (OPT2).md                    | IP, firewall, isolation                                  |
| 25    | 25 - WireGuard VPN (accès distant).md                  | Accès sécurisé depuis l’extérieur                        |
| 26    | 26 - Wazuh lab léger (Prodesk).md                      | SIEM de démonstration                                    |
| 27    | 27 - Clone système + préparation 2e machine.md         | Avant LUKS / machine dédiée                              |
| 28    | 28 - Activation LUKS Prodesk.md                        | Exécution de la checklist 17                             |

## Règles de documentation (à respecter)

- Toujours numéroter les fichiers dans l’ordre chronologique.
- Mettre une capture d’écran quand c’est utile.
- Expliquer chaque commande avec son effet.
- Mettre à jour ce fichier à chaque avancée importante.
- Garder une trace claire de ce qui est **terminé**, **en cours** ou **à faire**.

**Dernière mise à jour** : 18 mai 2026 – Hardening SSH + Fail2ban terminé, NetFlow configuré, BunkerWeb fonctionnel avec page personnalisée.

---
