# OPNsense Homelab Security

Infrastructure de sécurité réseau personnelle basée sur **OPNsense**.

Documentation technique complète d’un homelab durci (Firewall, IDS, WAF, segmentation réseau).

## Architecture

| Interface | Rôle | Réseau | Machine principale |
|-----------|------|--------|--------------------|
| **WAN** | Sortie Internet | 192.168.5.0/24 (FAI) | OPNsense |
| **LAN** | Clients de confiance | 10.0.0.0/24 | AlphaDeck |
| **OPT1** | Zone Bastion | 10.0.10.0/24 | Prodesk (BunkerWeb) |
| **OPT2** | Zone isolée / Lab | 10.0.20.0/24 | GX10 |

## Stack de sécurité

- **OPNsense** – Firewall + règles restrictives (Default Deny)
- **Suricata** – IDS passif (Emerging Threats + abuse.ch)
- **BunkerWeb** – WAF sur Prodesk
- **NetFlow / Insight** – Visibilité des flux
- **SSH** – Accès restreint depuis AlphaDeck uniquement

## Documentation

Toute la documentation se trouve dans le dossier [`docs/`](./docs/) :

| Fichier | Contenu |
|---------|---------|
| [01-architecture.md](./docs/01-architecture.md) | Vue d’ensemble de l’architecture |
| [02-network.md](./docs/02-network.md) | Schéma réseau |
| [03 - État actuel du Firewall.md](./docs/03%20-%20État%20actuel%20du%20Firewall.md) | État actuel du firewall |
| [04 - Firewall Rules Hardening & Cleanup.md](./docs/04%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md) | Règles firewall détaillées |
| [04b - VLANs et Segmentation Réseau.md](./docs/04b%20-%20VLANs%20et%20Segmentation%20Réseau.md) | VLANs et segmentation |
| [05 - État de la machine Prodesk.md](./docs/05%20-%20État%20de%20la%20machine%20Prodesk%20(Bastion%20Godmode).md) | État de la machine Prodesk |
| [06 - Aliases OPNsense.md](./docs/06%20-%20Aliases%20OPNsense.md) | Liste des aliases |
| [07 - Hardening du Bastion Godmode.md](./docs/07%20-%20Hardening%20du%20Bastion%20Godmode.md) | Philosophie & niveaux de hardening |
| [07b - Hardening du Bastion Godmode.md](./docs/07b%20-%20Hardening%20du%20Bastion%20Godmode.md) | Hardening technique (SSH, users, Fail2ban) |
| [07c - Pentest Externe Contrôlé.md](./docs/07c%20-%20Pentest%20Externe%20Contrôlé%20(Bastion%20Godmode).md) | Pentest externe contrôlé |
| [08 - Installation BunkerWeb.md](./docs/08%20-%20Installation%20BunkerWeb%20(Version%20Finale%20Fonctionnelle).md) | Installation BunkerWeb |
| [09 - Suricata.md](./docs/09%20-%20Suricata%20(Intrusion%20Detection)%20sur%20OPNsense.md) | Configuration Suricata |
| [10 - Accès SSH sécurisé.md](./docs/10%20-%20Accès%20SSH%20sécurisé%20depuis%20AlphaDeck.md) | Accès SSH sécurisé |
| [11 - BunkerWeb configuration finale.md](./docs/11%20-%20BunkerWeb%20configuration%20finale%20+%20virtual%20hosts%20+%20HTTPS%20Let’s%20Encrypt.md) | BunkerWeb + Let’s Encrypt |
| [11b - Port Forwarding.md](./docs/11b%20-%20Configuration%20Port%20Forwarding%20sur%20FAI%20Box.md) | Port Forwarding Box FAI |

## Objectif

Créer un environnement de production personnel **reproductible**, **documenté** et **durci**, inspiré des bonnes pratiques DevSecOps.

## Statut

- [x] Firewall OPNsense reconstruit et durci
- [x] Suricata opérationnel
- [x] BunkerWeb installé
- [ ] Multisite + Let’s Encrypt
- [ ] WireGuard
- [ ] Wazuh

---

> Projet personnel – Documentation en cours de structuration.
