# 01 – Architecture & Sommaire – Horus AIS / Bastion

**Projet** : Horus AIS – Infrastructure de défense locale  
**Date de création** : 15 mai 2026  
**Dernière mise à jour** : 15 août 2026  
**Objectif** : Documenter de façon structurée, reproductible et orientée portfolio professionnel une architecture de défense en profondeur derrière OPNsense.

Ce document est le point d’entrée du dépôt. Il décrit l’architecture réelle au 15 août 2026 et sert de sommaire vivant.

> **Note portfolio** : Les adresses IP internes sont présentées sous forme de placeholders (`10.0.10.x`, `10.0.10.y`, `10.0.0.z`). Les hostnames locaux sont généralisés. Les valeurs exactes du lab ne sont pas publiées.

---

## 1. Vision et principes

L’objectif n’est pas de construire un simple homelab confortable, mais une **architecture de défense en profondeur** (defense-in-depth) de niveau lab professionnel / data-center léger :

- **Prévention** : pare-feu default-deny, segmentation stricte, WAF, exposition minimale
- **Détection** : IDS (Suricata) + SIEM (Wazuh)
- **Protection des données** : chiffrement disque complet (LUKS) + déverrouillage distant contrôlé
- **Contrôle d’accès** : least privilege, SSH par clé uniquement, Fail2ban
- **Visibilité** : logs centralisés, agents, corrélation de base
- **Reproductibilité** : documentation exhaustive, orientée entretien et portfolio

Méthodologie inspirée des pratiques de documentation technique et de hardening (approche type lab cybersécurité / data center).

---

## 2. Topologie réseau actuelle (15 août 2026)

### 2.1 Vue d’ensemble

| Interface | Rôle | Réseau | Machine principale |
|-----------|------|--------|--------------------|
| **WAN** | Sortie Internet | Réseau FAI | OPNsense (HP EliteDesk 800 G2) |
| **LAN** | Clients de confiance | `10.0.0.0/24` | Workstation d’administration + postes locaux |
| **OPT1** | Zone Bastion | `10.0.10.0/24` | Bastion principal |
| **OPT2** | Zone isolée / Lab | `10.0.20.0/24` | Réservé (lab / futurs services isolés) |

Suricata fonctionne en mode IDS (passif) sur les interfaces pertinentes d’OPNsense.

### 2.2 Machines

| Rôle | Matériel | OS | Notes |
|------|----------|----|-------|
| **Firewall** | HP EliteDesk 800 G2 | OPNsense | Point unique de contrôle du trafic |
| **Bastion principal** | HP EliteDesk 800 G4 (i5 8ᵉ gen) | Rocky Linux 10 + LUKS | Services web, SIEM, tunnel, administration |
| **Ancien bastion** | Prodesk 600 Mini G2 (i3) | Debian 12 (éteint / backup) | Candidat honeypot ou lab secondaire |
| **Workstation admin** | Machine Windows | Windows 10 | Accès SSH, Dashboard Wazuh, agent SIEM actif |

**Adressage (placeholders lab / portfolio)** :

| Rôle | Adresse | Usage |
|------|---------|-------|
| Services (BunkerWeb, etc.) | `10.0.10.x/24` | Services exposés via le tunnel |
| Management / SSH / Wazuh | `10.0.10.y/24` | Administration et monitoring |
| Clients LAN | `10.0.0.z/24` | Workstation d’administration et postes de confiance |

![Schéma réseau local](../images/Schéma_Réseau_local.png)

---

## 3. Architecture de défense en profondeur

| Couche | Composants | Statut | Rôle |
|--------|------------|--------|------|
| **Périmètre** | OPNsense (default-deny, aliases, NetFlow) | ✅ | Contrôle strict du trafic entrant/sortant |
| **Détection réseau** | Suricata (IDS passif) | ✅ | Détection d’anomalies réseau |
| **Segmentation** | LAN / OPT1 / OPT2 | ✅ | Isolation des zones de confiance |
| **Protection des données** | LUKS full-disk + unlock distant (dracut-sshd) | ✅ | Chiffrement au repos + accès contrôlé au boot |
| **Contrôle d’accès** | SSH clé ED25519 uniquement + Fail2ban | ✅ | Least privilege + protection brute-force |
| **WAF / Reverse-proxy** | BunkerWeb (Docker, multisite) | ✅ | Protection des applications web |
| **Exposition** | Cloudflare Tunnel + cloudflared (utilisateur non-root) | ✅ | HTTPS sans aucun port ouvert sur le FAI |
| **SIEM** | Wazuh 4.14.x all-in-one | ✅ | Centralisation des logs + agent endpoint + syslog firewall |
| **Messagerie** | `@horus-ais.com` (SPF / DKIM / DMARC) | ✅ | Identité professionnelle |

**Niveau de maturité actuel : ~85-90 %** d’une architecture de défense de lab professionnel.

Les fondations sont opérationnelles. Les prochaines étapes visent la maturité « entretien / production légère ».

---

## 4. Stack technique détaillée (état au 15 août 2026)

### 4.1 Opérationnel

- **Firewall** : OPNsense – règles durcies, aliases, NetFlow / Insight
- **Bastion** : Rocky Linux 10 (écosystème RHEL-like, support long terme, excellent pour Docker + SIEM)
- **Chiffrement** : LUKS full disk + déverrouillage distant via dracut-sshd (point technique critique : sous BLS, `grubby` est obligatoire pour propager les arguments kernel)
- **Runtime** : Docker
- **WAF** : BunkerWeb multisite (`horus-ais.com` + sous-domaines)
- **Tunnel** : cloudflared exécuté sous utilisateur dédié non-root
- **Accès admin** : SSH (clés uniquement) + Fail2ban
- **SIEM** : Wazuh 4.14.x all-in-one + syslog OPNsense + agent sur la workstation d’administration
- **Messagerie** : domaine professionnel avec authentification SPF/DKIM/DMARC

### 4.2 En cours / prioritaires

| Élément | Priorité | Commentaire |
|---------|----------|-------------|
| WireGuard | Haute | Accès distant sécurisé et auditable |
| Logs BunkerWeb + Docker → Wazuh | Haute | Visibilité sur le trafic web réel et les attaques applicatives |
| Contenu réel des sites | Moyenne | Actuellement pages de test / branding |
| Backups & recovery testés | Haute | Surtout LUKS + configurations critiques |
| Suricata mode IPS (optionnel) | Basse | Actuellement IDS passif |
| Honeypot (ancien Prodesk) | Basse | Évolution future possible |

---

## 5. Évolution de l’architecture (contexte)

**Phase 1 (mai – juillet 2026)**  
Mise en place du socle sur l’ancien bastion (Prodesk 600 Mini G2 / Debian 12) : OPNsense, BunkerWeb, Cloudflare Tunnel, messagerie, hardening SSH de base.

**Phase 2 (août 2026)**  
Migration du rôle de **bastion principal** vers le HP EliteDesk 800 G4 sous Rocky Linux 10 :

- Meilleure capacité matérielle pour héberger simultanément BunkerWeb + Wazuh + Docker
- Base RHEL-like (crédibilité portfolio + packages officiels)
- LUKS activé dès l’installation + déverrouillage distant validé
- Documentation complète de la migration (fiches 23 à 32)

L’ancien Prodesk est conservé en backup et envisagé comme machine secondaire ou honeypot.

Cette migration a permis de valider un basculement propre de services critiques et de documenter les points d’attention spécifiques à Rocky Linux 10 + BLS (notamment l’usage obligatoire de `grubby` pour les arguments kernel early-boot).

---

## 6. Structure du dépôt

```
Opnsense-homelab-security/
├── docs/                  ← Documentation technique numérotée (runbooks)
├── images/                ← Captures d’écran et schémas
├── README.md              ← Vue d’ensemble portfolio
└── ...
```

---

## 7. Liste des documents (état au 15 août 2026)

| N° | Document | Statut | Contenu principal |
|----|----------|--------|-------------------|
| 01 | Architecture & Sommaire | ✅ À jour | Ce document (version portfolio) |
| 02 | Network | À enrichir / sanitizer | Interfaces OPNsense & plan d’adressage |
| 03 | État actuel du Firewall | ✅ Présent (à sanitizer) | Règles globales actuelles |
| 04 | Firewall Rules Hardening & Cleanup | ✅ Présent (à sanitizer) | Durcissement et nettoyage des règles |
| 04b | VLANs et Segmentation Réseau | ✅ Présent (à sanitizer) | Segmentation LAN / OPT1 / OPT2 |
| 05 | État de la machine Prodesk (ancien bastion) | Historique | Hardware & état de l’ancien bastion |
| 06 | Aliases OPNsense | ✅ Présent (à sanitizer) | Liste des aliases |
| 07 | Hardening du Bastion | ✅ Présent (à sanitizer) | Hardening initial |
| 07b | Hardening du Bastion Godmode | ✅ Présent (à sanitizer) | Version détaillée |
| 07c | Pentest Externe Contrôlé (Bastion Godmode) | ✅ Présent (à sanitizer) | Tests d’attaque contrôlés |
| 08 | Installation BunkerWeb (Version Finale Fonctionnelle) | ✅ Présent (à sanitizer) | Installation et config de base |
| 09 | Suricata (Intrusion Detection) sur OPNsense | ✅ Présent (à sanitizer) | Configuration IDS |
| 10 | Accès SSH sécurisé depuis AlphaDeck | ✅ Présent (à sanitizer) | Configuration SSH durcie (ancien bastion) |
| 11 | BunkerWeb configuration finale + virtual hosts + HTTPS Let’s Encrypt | Historique | Approche initiale (avant Cloudflare Tunnel) |
| 11b | Configuration Port Forwarding sur FAI Box | Historique | Remplacé par Cloudflare Tunnel |
| 12 | Domaine & Messagerie externe (horus-ais.com + OVH Zimbra) | ✅ Présent (à sanitizer) | Domaine et messagerie |
| 13 | Mise en service de la messagerie OVH Zimbra (horus-ais.com) | ✅ Présent (à sanitizer) | SPF / DKIM / DMARC |
| 14 | Inventaire Hardware & Ressources des machines | ✅ Présent (à sanitizer) | Inventaire matériel |
| 15 | Objectif du Cloudflare Tunnel | ✅ Présent (à sanitizer) | Pourquoi le tunnel |
| 16 | Installer cloudflared et Cloudflare Tunnel (Prodesk) | ✅ Présent (à sanitizer) | Installation initiale du tunnel |
| 17 | Checklist réinstallation Debian avec LUKS (Prodesk) | Historique | Checklist ancienne machine |
| 18 | Migration des règles Firewall OPNsense vers le nouveau système MVC_API | ✅ Présent (à sanitizer) | Migration des règles |
| 19 | Optimisation STUN_TURN et création de l'alias STUN_TURN_PORTS | ✅ Présent (à sanitizer) | Optimisation STUN/TURN |
| 20 | Hardening cloudflared - utilisateur dédié non-root | ✅ Présent (à sanitizer) | Durcissement du tunnel |
| 21 | Multisite BunkerWeb | ✅ Présent (à sanitizer) | Configuration multisite |
| 22 | Redirection mrdoolux.brave (Unstoppable) | En cours / à finaliser | Redirection Web3 |
| **23** | Rocky Linux G4 - install reseau et premier SSH | ✅ Terminé | Nouveau bastion principal |
| **24** | Users SSH et shell G4 | ✅ Terminé | Comptes et least privilege |
| **25** | Docker et migration cloudflared G4 | ✅ Terminé | Runtime + tunnel |
| **26** | Migration BunkerWeb multisite G4 | ✅ Terminé | WAF sur G4 |
| **27** | Déverrouillage distant LUKS via dracut-sshd (G4) | ✅ Terminé | Encryption + unlock distant |
| **28** | Durcissement SSH (G4) | ✅ Terminé | SSH clé-only |
| **29** | Fail2ban protection brute-force SSH (G4) | ✅ Terminé | Protection brute-force |
| **30** | Installation Wazuh SIEM (G4) | ✅ Terminé | SIEM all-in-one |
| **30a** | Paramétrage initial Wazuh (G4) | ✅ Terminé | Configuration initiale |
| **31** | Intégration Syslog OPNsense → Wazuh (G4) | ✅ Terminé | Centralisation logs firewall |
| **31A** | Filtres Discover & Alertes intelligentes OPNsense (G4) | ✅ Terminé | Filtres et alertes |
| **32** | Agent Wazuh sur AlphaDeck (Windows 10) | ✅ Terminé | Agent endpoint |

---

## 8. Règles de documentation

- Numéroter les fiches dans l’ordre chronologique d’avancement.
- Expliquer **chaque commande** et son effet dans le contexte de *cette* architecture.
- Ajouter des captures d’écran quand elles apportent de la clarté (sanitisées si nécessaire).
- Distinguer clairement : **Terminé** / **En cours** / **À faire**.
- Orientation **portfolio professionnel / data center / entretien** : on documente pour être relisible dans six mois ou pour un pair / recruteur.
- Mettre à jour ce fichier 01 à chaque jalon important.
- Version publique : généraliser systématiquement les éléments sensibles (IPs précises, hostnames locaux, identifiants).

---

## 9. Prochaines priorités (ordre logique)

1. **WireGuard** – accès distant sécurisé et auditable  
2. Ingestion des logs BunkerWeb + Docker dans Wazuh + règles d’alertes  
3. Contenu réel des sites + durcissement final  
4. Stratégie de backup / recovery testée (surtout LUKS + configurations critiques)  
5. (Optionnel) Suricata en mode IPS, honeypot sur l’ancien bastion, monitoring avancé  

---

## 10. Notes de version

- **Août 2026** : Migration complète du rôle de bastion principal vers le G4 (Rocky Linux 10 + LUKS).  
- Choix de Rocky Linux 10 : stabilité de type RHEL, support long terme, excellent écosystème pour Docker et SIEM.  
- Point technique critique retenu : sous BLS, `grubby` est obligatoire pour propager correctement les arguments kernel nécessaires au réseau early-boot (dracut-sshd).  
- Documentation désormais alignée sur un prisme professionnel (reproductibilité + portfolio).

---

**Document de référence – Architecture de défense Horus AIS**  
*Version portfolio – 15 août 2026*
