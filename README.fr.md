# OPNsense Homelab Security

Infrastructure de sécurité réseau personnelle entièrement documentée, construite autour d’**OPNsense**.

Homelab durci comprenant firewall, segmentation réseau, IDS, WAF, exposition web **sans ouverture de ports** (Cloudflare Tunnel), SIEM, chiffrement des disques, backups chiffrés automatisés et messagerie professionnelle.  
Documentation orientée reproductibilité, qualité portfolio et traçabilité technique.

---

## Architecture réseau

| Interface | Rôle | Réseau | Machine principale |
|-----------|------|--------|--------------------|
| **WAN** | Lien Internet | Réseau FAI | OPNsense (HP EliteDesk 800 G2) |
| **LAN** | Clients de confiance | 10.0.0.0/24 | AlphaDeck (poste d’administration) |
| **OPT1** | Zone bastion | 10.0.10.0/24 | **EliteDesk 800 G4** (Rocky Linux 10) |
| **OPT2** | Zone isolée / Lab | 10.0.20.0/24 | Réservée |

**Bastion principal (G4 – Rocky Linux 10)**  
- Services web (BunkerWeb) + Cloudflare Tunnel  
- SIEM Wazuh (Dashboard sur le port 8443)  
- Chiffrement LUKS + déverrouillage distant (dracut-sshd)  
- SSH durci + Fail2ban  
- Backups chiffrés automatisés avec rotation  

**Ancien bastion (Prodesk – Debian 12)**  
- Conservé comme référence / éventuel honeypot  

![Schéma du réseau local](./images/Schéma_Réseau_local.png)

---

## Stack de sécurité

| Composant | Rôle | Statut |
|-----------|------|--------|
| **OPNsense** | Firewall default-deny, aliases, segmentation | ✅ |
| **Suricata** | IDS passif (EVE JSON) | ✅ |
| **BunkerWeb** | WAF / reverse-proxy (Docker), multisite | ✅ |
| **Cloudflare Tunnel** | Exposition HTTPS sans ouverture de ports | ✅ |
| **cloudflared** | Client tunnel en non-root | ✅ |
| **Messagerie** | `@horus-ais.com` (OVH Zimbra, SPF/DKIM/DMARC) | ✅ |
| **SSH** | Accès restreint, clés uniquement, Fail2ban | ✅ |
| **LUKS** | Chiffrement complet + déverrouillage distant | ✅ |
| **Wazuh** | SIEM all-in-one (Indexer + Manager + Dashboard) | ✅ |
| **Syslog OPNsense → Wazuh** | Collecte filterlog / Suricata | ✅ |
| **Agent Wazuh** | AlphaDeck (Windows 10) | ✅ |
| **Logs BunkerWeb / Docker → Wazuh** | Collecte + décodage Bad Behavior | ✅ |
| **WireGuard** | Accès VPN distant (Road Warrior) | ✅ |
| **Backups chiffrés** | Rotation Daily + Weekly sur volume LUKS | ✅ |

### Sites exposés (via tunnel)

| Hostname | Backend | Contenu |
|----------|---------|---------|
| `horus-ais.com` / `www` | BunkerWeb | Site principal |
| `mrdoolux.horus-ais.com` | BunkerWeb | Tests / branding |
| `mrdoolux.brave` | Redirection Web3 | Configurée |

---

## Documentation

Dossier [`docs/`](./docs/) — captures d’écran dans [`images/`](./images/).

> **Note :** La plupart des fiches techniques sont actuellement en français. Une version anglaise des documents clés sera ajoutée progressivement.

### Socle
- [01 – Architecture](./docs/01-architecture.md)
- [02 – Réseau](./docs/02-network.md)
- [03 – État du firewall](./docs/03-etat-firewall.md)
- [04 – Durcissement des règles firewall](./docs/04-Firewall-Rules-Hardening.md)
- [04b – VLANs](./docs/04b-VLANs-Segmentation.md)
- [05 – État de la machine Prodesk](./docs/05-Etat-machine-Prodesk.md)
- [06 – Aliases OPNsense](./docs/06-Aliases-OPNsense.md)
- [07 / 07b – Durcissement du bastion](./docs/07-Hardening-Bastion.md)
- [07c – Pentest externe contrôlé](./docs/07c-Pentest-Externe-Controle.md)
- [08 – Installation BunkerWeb](./docs/08-Installation-BunkerWeb.md)
- [09 – Suricata](./docs/09-Suricata-OPNsense.md)
- [10 – Accès SSH sécurisé](./docs/10-Acces-SSH-securise.md)

### Exposition web
- [11 / 11b – Approche historique (port-forward)](./docs/11-BunkerWeb-config-finale.md)
- [15 – Objectif Cloudflare Tunnel](./docs/15-Objectif-Cloudflare-Tunnel.md)
- [16 – Installation cloudflared](./docs/16-Installer-cloudflared.md)
- [20 – Durcissement cloudflared non-root](./docs/20-Hardening-cloudflared.md)
- [21 – BunkerWeb multisite](./docs/21-Multisite-BunkerWeb.md)
- [22 – Redirection Web3](./docs/22-Redirection-mrdoolux-brave.md)
- [40 – Sites en 403 / Real IP Cloudflare](./docs/40%20-%20Sites-en-403-Analyse-et-resolution-Real-IP-Cloudflare-SANITIZED-2026-08-19%20(1).md)

### Messagerie, hardware, opérations
- [12 – Domaine & messagerie](./docs/12-Domaine-Messagerie-externe.md)
- [14 – Inventaire hardware](./docs/14-Inventaire-Hardware.md)
- [17 – Checklist LUKS](./docs/17-Checklist-LUKS.md)
- [18 – Migration règles firewall MVC](./docs/18-Migration-Firewall-MVC-API.md)
- [19 – Optimisation STUN/TURN](./docs/19-Optimisation-STUN-TURN.md)

### Migration du bastion vers G4 (Rocky Linux)
- [23 – Installation réseau + premier SSH](./docs/23%20-%20Rocky%20Linux%20G4%20-%20install%20reseau%20et%20premier%20SSH.md)
- [24 – Utilisateurs, SSH et shell](./docs/24%20-%20Users%20SSH%20et%20shell%20G4.md)
- [25 – Docker + migration cloudflared](./docs/25%20-%20Docker%20et%20migration%20cloudflared%20G4.md)
- [26 – Migration BunkerWeb multisite](./docs/26%20-%20Migration%20BunkerWeb%20multisite%20G4.md)
- [27 – Déverrouillage distant LUKS](./docs/27%20-%20Déverrouillage%20distant%20LUKS%20via%20dracut-sshd%20(G4).md)
- [28 – Durcissement SSH](./docs/28%20-%20Durcissement%20SSH%20(G4).md)
- [29 – Fail2ban](./docs/29%20-%20Fail2ban%20protection%20brute-force%20SSH%20(G4).md)
- [30 – Installation Wazuh SIEM](./docs/30%20-%20Installation%20Wazuh%20SIEM%20(G4).md)
- [30a – Paramétrage initial Wazuh](./docs/30a%20-%20Paramétrage%20initial%20Wazuh%20(G4).md)
- [31 – Syslog OPNsense → Wazuh](./docs/31%20-%20Intégration%20Syslog%20OPNsense%20→%20Wazuh%20(G4).md)
- [31A – Filtres Discover & alertes intelligentes](./docs/31A%20-%20Filtres%20Discover%20%26%20Alertes%20intelligentes%20OPNsense%20(G4).md)
- [32 – Agent Wazuh sur AlphaDeck](./docs/32%20-%20Agent%20Wazuh%20sur%20AlphaDeck%20(Windows%2010).md)

### Backups, VPN & Monitoring avancé
- [33 – Incident boot / journal persistant](./docs/33-Incident-Boot-Recovery-Journal-Persistant-BIOS%20(1).md)
- [34 – Premier backup complet (externe)](./docs/34%20–%20Premier%20backup%20complet%20de%20la%20G4%20(Rocky%20Linux)%20vers%20disque%20externe.md)
- [35 – Volume de backup interne LUKS](./docs/35%20-%20Volume%20de%20backup%20interne%20chiffré%20(LUKS)%20sur%20le%20SSD%20Verbatim%201%20To.md)
- [36 – Automatisation des backups](./docs/36%20–%20Automatisation%20des%20backups%20G4%20+%20OPNsense.md)
- [36A – Check-up des backups](./docs/36A-Check-up-backups-G4-OPNsense.md)
- [36B – Rotation et rétention des backups](./docs/36B%20-%20Rotation%20et%20rétention%20des%20backups%20G4%20(Daily_Weekly).md)
- [37 – Maintenance G4 (correctif multi-LUKS)](./docs/37-Maintenance-G4-17-aout-2026-Upgrade-Correctif-multi-LUKS-BunkerWeb.md)
- [38 – WireGuard VPN](./docs/38%20-%20WireGuard-VPN-OPNsense-2026-08-18.md)
- [39 – Collecte logs BunkerWeb / Docker → Wazuh](./docs/39%20-%20Collecte-logs-BunkerWeb-Docker-Wazuh-SANITIZED-2026-08-19.md)
- [39A – Décodeurs Bad Behavior](./docs/39A%20-%20BunkerWeb-BadBehavior-Wazuh-Decoders-Rules-2026-08-20.md)
- [39B – Règles PCRE2 enrichies](./docs/39B%20-%20Decoder-PCRE2-BadBehavior-Regle-enrichie-Wazuh-2026-08-20.md)

---

## Objectif

Construire un **bastion homelab reproductible, documenté et défendable** selon une approche laboratoire cybersécurité / DevSecOps : chaque étape est tracée (commandes, effets, captures d’écran, décisions).

---

## État global (20 août 2026)

- [x] Firewall OPNsense reconstruit et durci  
- [x] Suricata opérationnel  
- [x] BunkerWeb + multisite  
- [x] Cloudflare Tunnel + durcissement `cloudflared` non-root  
- [x] Messagerie professionnelle `@horus-ais.com`  
- [x] Migration du bastion vers EliteDesk G4 (Rocky Linux)  
- [x] LUKS + déverrouillage distant  
- [x] SSH durci + Fail2ban  
- [x] SIEM Wazuh all-in-one  
- [x] Syslog OPNsense → Wazuh  
- [x] Agent Wazuh sur le poste d’administration  
- [x] Ingestion des logs BunkerWeb / Docker dans Wazuh  
- [x] WireGuard VPN (Road Warrior) opérationnel  
- [x] Backups chiffrés + rotation daily/weekly  
- [ ] Contenu réel des sites  
- [ ] Alertes email (Fail2ban / Wazuh / OPNsense)  

---

> Projet personnel de laboratoire cybersécurité — documentation axée sur la clarté et la reproductibilité.  
> Dépôt : [Lu-Suy/Opnsense-homelab-security](https://github.com/Lu-Suy/Opnsense-homelab-security)

**English version available here:** [README.md](./README.md)


