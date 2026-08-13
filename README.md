# OPNsense Homelab Security

Infrastructure de sécurité réseau personnelle basée sur **OPNsense**, documentée de bout en bout.

Homelab durci : firewall, segmentation, IDS, WAF, exposition web **sans port-forwarding** (Cloudflare Tunnel), messagerie pro. Documentation orientée reproduction, jury et portfolio cybersécurité.

---

## Architecture réseau

| Interface | Rôle | Réseau | Machine principale |
|-----------|------|--------|--------------------|
| **WAN** | Sortie Internet | 192.168.5.0/24 (FAI) | OPNsense (HP EliteDesk 800 G2) |
| **LAN** | Clients de confiance | 10.0.0.0/24 | AlphaDeck |
| **OPT1** | Zone Bastion | 10.0.10.0/24 | Prodesk 600 Mini G2 (Debian 12) |
| **OPT2** | Zone isolée / Lab | 10.0.20.0/24 | GX10 (prévu) |

**Bastion (Prodesk)**  
- Services web : `10.0.10.10` (BunkerWeb)  
- Admin SSH : `10.0.10.11` (clés ED25519, Fail2ban)

![Schéma réseau local](./images/Schéma_Réseau_local.png)

---

## Stack de sécurité (état août 2026)

| Composant | Rôle | Statut |
|-----------|------|--------|
| **OPNsense** | Firewall default-deny, aliases, NetFlow | ✅ |
| **Suricata** | IDS passif (Emerging Threats + abuse.ch) | ✅ |
| **BunkerWeb** | WAF / reverse-proxy (Docker), **multisite** | ✅ |
| **Cloudflare Tunnel** | Exposition HTTPS sans ouvrir de ports | ✅ |
| **cloudflared** | Client tunnel sous utilisateur **non-root** | ✅ |
| **Messagerie** | `@horus-ais.com` (OVH Zimbra, SPF/DKIM/DMARC) | ✅ |
| **SSH** | Accès restreint depuis le LAN (AlphaDeck) | ✅ |
| **WireGuard** | VPN distant | ⏳ Prévu |
| **Wazuh** | SIEM léger | ⏳ Prévu |
| **LUKS** | Chiffrement disque Prodesk | ⏳ Checklist prête |

### Sites exposés (via tunnel)

| Hostname | Backend | Contenu |
|----------|---------|---------|
| `horus-ais.com` / `www` | BunkerWeb | Page principale |
| `mrdoolux.horus-ais.com` | BunkerWeb | Tests |
| `mrdoolux.brave` (Web3) | Redirection → Web2 | En cours (Unstoppable) |

---

## Choix d’architecture : Cloudflare Tunnel

**Approche retenue :** Cloudflare Tunnel (`cloudflared` sur la Prodesk).

- Aucun port 80/443 ouvert sur la box FAI ni sur OPNsense  
- IP publique non exposée  
- TLS terminé côté Cloudflare  

**Approche historique (docs 11 / 11b) :** port-forwarding + Let’s Encrypt local.  
Réalisable, mais plus exposée. Conservée comme **méthode alternative / pédagogique**.

---

## Documentation

Dossier [`docs/`](./docs/) — captures dans [`images/`](./images/).

### Socle

| Fichier | Contenu |
|---------|---------|
| [01 – Architecture](./docs/01-architecture.md) | Vue d’ensemble |
| [02 – Network](./docs/02-network.md) | Réseau |
| [03 – État Firewall](./docs/03%20-%20État%20actuel%20du%20Firewall.md) | État firewall |
| [04 – Hardening règles](./docs/04%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md) | Règles détaillées |
| [04b – VLANs](./docs/04b%20-%20VLANs%20et%20Segmentation%20Réseau.md) | Segmentation |
| [05 – Prodesk](./docs/05%20-%20État%20de%20la%20machine%20Prodesk%20(Bastion%20Godmode).md) | État bastion |
| [06 – Aliases](./docs/06%20-%20Aliases%20OPNsense.md) | Aliases |
| [07 / 07b – Hardening](./docs/07b%20-%20Hardening%20du%20Bastion%20Godmode.md) | Hardening bastion |
| [07c – Pentest](./docs/07c%20-%20Pentest%20Externe%20Contrôlé%20(Bastion%20Godmode).md) | Tests contrôlés |
| [08 – BunkerWeb](./docs/08%20-%20Installation%20BunkerWeb%20(Version%20Finale%20Fonctionnelle).md) | Installation WAF |
| [09 – Suricata](./docs/09%20-%20Suricata%20(Intrusion%20Detection)%20sur%20OPNsense.md) | IDS |
| [10 – SSH](./docs/10%20-%20Accès%20SSH%20sécurisé%20depuis%20AlphaDeck.md) | SSH durci |

### Exposition web

| Fichier | Contenu | Note |
|---------|---------|------|
| [11 – BunkerWeb + LE](./docs/11%20-%20BunkerWeb%20configuration%20finale%20+%20virtual%20hosts%20+%20HTTPS%20Let’s%20Encrypt.md) | Virtual hosts + Let’s Encrypt | Historique |
| [11b – Port forwarding](./docs/11b%20-%20Configuration%20Port%20Forwarding%20sur%20FAI%20Box.md) | NAT box FAI | Historique |
| [15 – Objectif Tunnel](./docs/15%20-%20Objectif%20du%20Cloudflare%20Tunnel.md) | Pourquoi le tunnel | ✅ |
| [16 – cloudflared](./docs/16%20-%20Installer%20cloudflared%20et%20Cloudflare%20Tunnel%20(Prodesk).md) | Install + service | ✅ |
| [20 – Hardening cloudflared](./docs/20%20-%20Hardening%20cloudflared%20-%20utilisateur%20dédié%20non-root.md) | User non-root | ✅ |
| [21 – Multisite](./docs/21%20-%20Multisite%20BunkerWeb.md) | Multisite BunkerWeb | ✅ |
| [22 – Web3 redirect](./docs/22%20-%20Redirection%20mrdoolux.brave%20(Unstoppable).md) | `mrdoolux.brave` | ⏳ |

### Messagerie, hardware, ops

| Fichier | Contenu |
|---------|---------|
| [12 – Domaine & mail](./docs/12%20-%20Domaine%20%26%20Messagerie%20externe%20(horus-ais.com%20+%20OVH%20Zimbra).md) | Domaine + OVH |
| [13 – Zimbra en service](./docs/13%20-%20Mise%20en%20service%20de%20la%20messagerie%20OVH%20Zimbra%20(horus-ais.com).md) | SPF / DKIM / DMARC |
| [14 – Inventaire](./docs/14%20-%20Inventaire%20Hardware%20%26%20Ressources%20des%20machines%20(1).md) | Hardware |
| [17 – Checklist LUKS](./docs/17%20-%20Checklist%20réinstallation%20Debian%20avec%20LUKS%20(Prodesk).md) | Réinstall chiffrée |
| [18 – Firewall MVC](./docs/18%20-%20Migration%20des%20règles%20Firewall%20OPNsense%20vers%20le%20nouveau%20système%20MVC_API.md) | Migration règles |
| [19 – STUN/TURN](./docs/19%20-%20Optimisation%20STUN_TURN%20et%20création%20de%20l'alias%20STUN_TURN_PORTS.md) | Alias STUN/TURN |

---

## Objectif

Construire un **bastion homelab reproductible**, documenté et défendable, dans une démarche type lab cybersécurité / DevSecOps : chaque étape tracée (commandes, effets, captures, décisions).

---

## Statut global (août 2026)

- [x] Firewall OPNsense reconstruit et durci  
- [x] Suricata opérationnel  
- [x] BunkerWeb + multisite  
- [x] Cloudflare Tunnel + hardening `cloudflared` non-root  
- [x] Messagerie `@horus-ais.com`  
- [ ] Contenu réel des sites  
- [ ] Finalisation redirection Web3  
- [ ] WireGuard  
- [ ] Wazuh (lab léger)  
- [ ] LUKS Prodesk  
- [ ] GX10 (OPT2)  

---

> Projet personnel de lab cybersécurité — documentation orientée clarté et reproductibilité.  
> Dépôt portfolio : [Lu-Suy/Opnsense-homelab-security](https://github.com/Lu-Suy/Opnsense-homelab-security)
