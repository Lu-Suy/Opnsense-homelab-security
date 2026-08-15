# OPNsense Homelab Security

Infrastructure de sécurité réseau personnelle basée sur **OPNsense**, documentée de bout en bout.

Homelab durci : firewall, segmentation, IDS, WAF, exposition web **sans port-forwarding** (Cloudflare Tunnel), messagerie professionnelle.  
Documentation orientée reproduction, portfolio et suivi technique.

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
- Administration SSH : `10.0.10.11` (clés ED25519, Fail2ban)

![Schéma réseau local](./images/Schéma_Réseau_local.png)

---

## Stack de sécurité

| Composant | Rôle | Statut |
|-----------|------|--------|
| **OPNsense** | Firewall default-deny, aliases, NetFlow | ✅ |
| **Suricata** | IDS passif | ✅ |
| **BunkerWeb** | WAF / reverse-proxy (Docker), multisite | ✅ |
| **Cloudflare Tunnel** | Exposition HTTPS sans ouvrir de ports | ✅ |
| **cloudflared** | Client tunnel sous utilisateur non-root | ✅ |
| **Messagerie** | `@horus-ais.com` (OVH Zimbra, SPF/DKIM/DMARC) | ✅ |
| **SSH** | Accès restreint depuis le LAN | ✅ |
| **WireGuard** | VPN distant | ⏳ |
| **Wazuh** | SIEM léger | ⏳ |
| **LUKS** | Chiffrement disque Prodesk | ⏳ |

### Sites exposés (via tunnel)

| Hostname | Backend | Contenu |
|----------|---------|---------|
| `horus-ais.com` / `www` | BunkerWeb | Site principal |
| `mrdoolux.horus-ais.com` | BunkerWeb | Tests / branding |
| `mrdoolux.brave` | Redirection Web3 | En cours |

---

## Documentation

Dossier [`docs/`](./docs/) — captures dans [`images/`](./images/).

### Socle
- [01 – Architecture](./docs/01-architecture.md)
- [02 – Network](./docs/02-network.md)
- [03 – État Firewall](./docs/03%20-%20État%20actuel%20du%20Firewall.md)
- [04 – Hardening règles](./docs/04%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md)
- [04b – VLANs](./docs/04b%20-%20VLANs%20et%20Segmentation%20Réseau.md)
- [05 – Prodesk / Bastion](./docs/05%20-%20État%20de%20la%20machine%20Prodesk%20(Bastion%20Godmode).md)
- [06 – Aliases](./docs/06%20-%20Aliases%20OPNsense.md)
- [07 / 07b – Hardening Bastion](./docs/07b%20-%20Hardening%20du%20Bastion%20Godmode.md)
- [07c – Pentest contrôlé](./docs/07c%20-%20Pentest%20Externe%20Contrôlé%20(Bastion%20Godmode).md)
- [08 – BunkerWeb](./docs/08%20-%20Installation%20BunkerWeb%20(Version%20Finale%20Fonctionnelle).md)
- [09 – Suricata](./docs/09%20-%20Suricata%20(Intrusion%20Detection)%20sur%20OPNsense.md)
- [10 – SSH](./docs/10%20-%20Accès%20SSH%20sécurisé%20depuis%20AlphaDeck.md)

### Exposition web
- [11 / 11b – Approche historique (port-forward + LE)](./docs/11%20-%20BunkerWeb%20configuration%20finale%20+%20virtual%20hosts%20+%20HTTPS%20Let’s%20Encrypt.md)
- [15 – Objectif Cloudflare Tunnel](./docs/15%20-%20Objectif%20du%20Cloudflare%20Tunnel.md)
- [16 – Installation cloudflared](./docs/16%20-%20Installer%20cloudflared%20et%20Cloudflare%20Tunnel%20(Prodesk).md)
- [20 – Hardening cloudflared non-root](./docs/20%20-%20Hardening%20cloudflared%20-%20utilisateur%20dédié%20non-root.md)
- [21 – Multisite BunkerWeb](./docs/21%20-%20Multisite%20BunkerWeb.md)
- [22 – Redirection Web3](./docs/22%20-%20Redirection%20mrdoolux.brave%20(Unstoppable).md)

### Messagerie, hardware, ops
- [12 / 13 – Domaine & messagerie](./docs/12%20-%20Domaine%20%26%20Messagerie%20externe%20(horus-ais.com%20+%20OVH%20Zimbra).md)
- [14 – Inventaire hardware](./docs/14%20-%20Inventaire%20Hardware%20%26%20Ressources%20des%20machines%20(1).md)
- [17 – Checklist LUKS](./docs/17%20-%20Checklist%20réinstallation%20Debian%20avec%20LUKS%20(Prodesk).md)
- [18 – Migration règles MVC](./docs/18%20-%20Migration%20des%20règles%20Firewall%20OPNsense%20vers%20le%20nouveau%20système%20MVC_API.md)
- [19 – STUN/TURN](./docs/19%20-%20Optimisation%20STUN_TURN%20et%20création%20de%20l'alias%20STUN_TURN_PORTS.md)

---

## Objectif

Construire un **bastion homelab reproductible**, documenté et défendable, dans une démarche lab cybersécurité / DevSecOps : chaque étape tracée (commandes, effets, captures, décisions).

---

## Statut global

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
> Dépôt : [Lu-Suy/Opnsense-homelab-security](https://github.com/Lu-Suy/Opnsense-homelab-security)
