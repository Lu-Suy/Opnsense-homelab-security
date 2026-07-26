# 03 - État actuel du Firewall

**Dernière mise à jour** : Juillet 2026  
**Version OPNsense** : 26.1.x

## Philosophie générale

- **Default Deny** sur toutes les interfaces
- Règles explicites + logs activés
- Segmentation forte entre les zones (LAN / OPT1 / OPT2)
- Accès admin ultra-restreint

## État par interface

### WAN
- Règles d’urgence (anti-lockout) depuis une IP précise du réseau FAI (192.168.5.177)
- Ports 22 et 443 ouverts uniquement pour cette IP
- Tout le reste bloqué par défaut

### LAN (10.0.0.0/24)
- Accès BunkerWeb (80/443) depuis AlphaDeck uniquement
- SSH vers Prodesk Manager (10.0.10.11) uniquement depuis AlphaDeck
- DNS (53, 853, 443), NTP, HTTP/HTTPS outbound autorisés
- STUN/TURN + ports WebRTC pour Grok Voice
- **Block All** en dernière règle

### OPT1 – Bastion (10.0.10.0/24)
- DNS (Do53 + DoT + DoH)
- NTP
- HTTP/HTTPS outbound
- SSH uniquement depuis AlphaDeck vers 10.0.10.11
- Blocage du trafic vers les réseaux privés (anti-pivot)
- **Block All** en dernière règle

### OPT2 – GX10 (10.0.20.0/24)
- DNS + NTP + HTTP/HTTPS outbound
- SSH depuis AlphaDeck uniquement (optionnel)
- Blocage fort vers les réseaux privés
- **Block All** en dernière règle (isolation forte)

## Voir aussi

- [04b - Firewall Rules Hardening & Cleanup](./04b%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md) → Détail complet des règles
- [06 - Aliases OPNsense](./06%20-%20Aliases%20OPNsense.md)
- [09 - Suricata](./09%20-%20Suricata%20(Intrusion%20Detection)%20sur%20OPNsense.md)
