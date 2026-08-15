# 02 – Configuration Réseau & Segmentation

**Projet** : Horus AIS – Infrastructure de défense locale  
**Date de création** : mai 2026  
**Dernière mise à jour** : 15 août 2026  
**Objectif** : Décrire le plan d’adressage, les interfaces OPNsense et les choix de segmentation réseau. Expliquer *pourquoi* chaque zone existe et comment le routing est maîtrisé.

> **Note portfolio** : Les adresses IP internes sont présentées sous forme de placeholders (`10.0.0.0/24`, `10.0.10.x`, `10.0.10.y`, `10.0.20.0/24`). Les valeurs exactes du lab ne sont pas publiées.

---

## 1. Principes de design réseau

L’architecture réseau repose sur trois principes simples mais stricts :

1. **Segmentation stricte**  
   Chaque type de charge de travail vit dans sa propre zone. On ne mélange jamais les clients de confiance, les services exposés et les machines de lab.

2. **Least privilege réseau**  
   Une machine n’a accès qu’à ce dont elle a réellement besoin. Les flux inter-zones sont filtrés par OPNsense (default-deny).

3. **Séparation des plans (services vs management)**  
   Sur le bastion, l’IP des services web n’est pas la même que l’IP d’administration. Cela permet des règles firewall beaucoup plus fines et limite la surface d’attaque en cas de compromission d’un service.

Ces choix sont inspirés des pratiques de défense en profondeur (defense-in-depth) et de zero-trust light applicables à un lab professionnel.

---

## 2. Pourquoi quatre interfaces sur OPNsense ?

OPNsense dispose de quatre interfaces physiques utilisées de la façon suivante :

| Interface | Rôle | Réseau | Justification |
|-----------|------|--------|---------------|
| **WAN** | Sortie Internet | Réseau FAI | Seule interface connectée au fournisseur d’accès. Tout le trafic sortant et entrant (via le tunnel) passe par ici. |
| **LAN** | Zone de confiance | `10.0.0.0/24` | Contient les postes d’administration et les machines de confiance. C’est la zone la plus « humaine ». |
| **OPT1** | Zone Bastion | `10.0.10.0/24` | Zone isolée dédiée au bastion principal (services web, SIEM, tunnel, administration). Isolation forte par rapport au LAN. |
| **OPT2** | Zone isolée / Lab | `10.0.20.0/24` | Zone de test, futur honeypot ou services expérimentaux. Blast radius limité : une compromission ici n’impacte pas le bastion ni les clients. |

### Pourquoi cette séparation est importante

- **WAN** isolé → on contrôle précisément ce qui sort et ce qui rentre.
- **LAN** séparé du bastion → un poste utilisateur compromis ne peut pas pivoter directement vers les services critiques sans passer par les règles firewall.
- **OPT1 (Bastion)** isolé → les services exposés (même derrière Cloudflare Tunnel) ne partagent pas le même segment que les postes de travail.
- **OPT2** → permet d’expérimenter (honeypot, lab AI, tests de sécurité) sans risque pour le reste de l’infrastructure.

C’est la base de la **segmentation réseau** de l’architecture de défense.

---

## 3. Plan d’adressage du Bastion (OPT1)

Le bastion principal (actuellement HP EliteDesk 800 G4 sous Rocky Linux 10) utilise un **dual-IP** :

| Rôle | Adresse (placeholder) | Usage | Justification |
|------|-----------------------|-------|---------------|
| **Services** | `10.0.10.x/24` | BunkerWeb, services web, reverse-proxy | IP « publique » interne des applications. C’est celle vers laquelle le Cloudflare Tunnel pointe. |
| **Management** | `10.0.10.y/24` | SSH, Wazuh Dashboard (port 8443), administration | IP réservée à l’administration. On peut interdire ou restreindre très fortement l’accès à cette IP depuis les autres zones. |

### Pourquoi le dual-IP ?

1. **Séparation des rôles**  
   Si BunkerWeb est un jour compromis, l’attaquant se trouve sur l’IP services. L’IP management (SSH + Dashboard Wazuh) reste sur un autre adressage et peut être protégée par des règles plus strictes.

2. **Règles firewall plus fines**  
   On peut autoriser le tunnel Cloudflare uniquement vers l’IP services, et n’autoriser SSH / Wazuh Dashboard que depuis le LAN (workstation d’administration).

3. **Clarté opérationnelle**  
   On sait immédiatement si un flux concerne les services exposés ou l’administration.

Ce modèle dual-IP a été conservé lors de la migration de l’ancien bastion (Prodesk) vers le G4.

---

## 4. Routing et flux inter-zones

### 4.1 Gateway du bastion (état actuel)

Le bastion (OPT1) a comme gateway par défaut l’interface OPT1 d’OPNsense (`10.0.10.1`).  
Tout le trafic sortant du bastion (mises à jour, accès Internet contrôlé, etc.) passe donc par OPNsense.  
Aucune route statique supplémentaire n’a été ajoutée sur le G4 lors de la migration.

### 4.2 Comment les zones communiquent

OPNsense assure nativement le routage entre ses interfaces (LAN ↔ OPT1 ↔ OPT2 ↔ WAN).  
Cependant, **le routage seul ne décide de rien** : toutes les communications inter-zones sont soumises aux règles firewall (default-deny + aliases).

Les flux utiles (exemples) :
- Workstation admin (LAN) → Bastion management (`10.0.10.y`) : SSH + Wazuh Dashboard (port 8443)
- Bastion → WAN : sorties contrôlées (mises à jour, Cloudflare Tunnel…)
- OPNsense → Bastion : syslog
- Agent Wazuh (LAN) → Manager Wazuh (OPT1)

Tout autre flux est bloqué par défaut.

### 4.3 Note historique sur le « routing »

Dans les toutes premières phases du projet (ancien bastion Prodesk), une route statique avait été mise en place pour permettre la communication entre le LAN et OPT1.  
À ce moment-là, les règles firewall n’étaient pas encore complètement écrites et durcies. Le routing a donc servi de **contournement temporaire** pour faire communiquer les machines en attendant que les règles soient en place.

Une fois les règles firewall correctement configurées (default-deny + sources restreintes), ce contournement n’est plus nécessaire.  
Le contrôle réel des flux repose désormais entièrement sur les règles OPNsense, pas sur des routes statiques.

### 4.4 OPT2

La zone OPT2 est volontairement plus isolée. Les flux vers/depuis OPT2 sont encore plus restreints (ou absents) selon les besoins du lab / honeypot.

---

## 5. Évolution de l’adressage

| Période | Bastion principal | Plan d’adressage |
|---------|-------------------|------------------|
| Mai – juillet 2026 | Prodesk 600 Mini G2 (Debian 12) | Dual-IP 10.0.10.x / 10.0.10.y |
| Août 2026 → | HP EliteDesk 800 G4 (Rocky Linux 10) | **Même plan d’adressage** conservé |

Le plan d’adressage n’a pas changé lors de la migration. Seule la machine physique et l’OS ont changé. Cela a permis une migration plus propre des services (BunkerWeb, cloudflared, etc.).

---

## 6. Résumé des zones et niveaux de confiance

| Zone | Niveau de confiance | Contenu typique | Exposition |
|------|---------------------|-----------------|----------|
| WAN | Non fiable | Internet | Contrôlée par OPNsense + Cloudflare Tunnel |
| LAN | Élevé | Workstation admin, postes locaux | Interne uniquement |
| OPT1 (Bastion) | Élevé (mais isolé) | Services + management | Services via Tunnel uniquement |
| OPT2 | Faible / expérimental | Lab, honeypot potentiel | Très restreinte |

---

## 7. Voir aussi

- [01 – Architecture & Sommaire](./01-architecture.md)
- [04 – Firewall Rules Hardening & Cleanup](./04%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md)
- [04b – VLANs et Segmentation Réseau](./04b%20-%20VLANs%20et%20Segmentation%20Réseau.md)
- [06 – Aliases OPNsense](./06%20-%20Aliases%20OPNsense.md)
- [23 – Rocky Linux G4 – install réseau et premier SSH](./23%20-%20Rocky%20Linux%20G4%20-%20install%20reseau%20et%20premier%20SSH.md)

---

## 8. Notes de version

- **Août 2026** : Document enrichi pour expliquer explicitement les choix de segmentation, le dual-IP et le routing.  
- Alignement avec le bastion actuel (G4 Rocky Linux 10).  
- Sanitization des adresses IP pour la version portfolio.

---
![Schéma réseau local](../images/Schéma_Réseau_local.png)
**Document de référence – Plan réseau & segmentation**  
*Version portfolio – 15 août 2026*
