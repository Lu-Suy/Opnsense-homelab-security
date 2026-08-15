# 03 – État actuel du Firewall

**Document historique – Snapshot**  
**Date du snapshot** : Juillet 2026  
**Version OPNsense** : 26.1.x  
**Dernière mise à jour du document** : 15 août 2026 (sanitization portfolio)

> **Note importante** :  
> Ce document est un **snapshot historique** de l’état du firewall en juillet 2026, pendant la phase où le bastion principal était encore le Prodesk (Debian 12).  
> L’architecture et les règles ont évolué depuis (migration vers le G4 Rocky Linux 10, durcissement supplémentaire, intégration Wazuh, etc.).  
> Pour l’état et les règles actuels, se référer aux documents 04, 04b, 18 et aux fiches de migration G4 (23+).

---

## Philosophie générale (juillet 2026)

- **Default Deny** sur toutes les interfaces
- Règles explicites + logs activés
- Segmentation forte entre les zones (LAN / OPT1 / OPT2)
- Accès admin ultra-restreint

---

## État par interface (snapshot juillet 2026)

### WAN
- Règles d’urgence (anti-lockout) depuis une IP précise du réseau FAI
- Ports 22 et 443 ouverts uniquement pour cette IP de confiance
- Tout le reste bloqué par défaut

### LAN (`10.0.0.0/24`)
- Accès BunkerWeb (80/443) depuis la workstation d’administration uniquement
- SSH vers le bastion (IP management) uniquement depuis la workstation d’administration
- DNS (53, 853, 443), NTP, HTTP/HTTPS outbound autorisés
- STUN/TURN + ports WebRTC (pour usage voix)
- **Block All** en dernière règle

### OPT1 – Zone Bastion (`10.0.10.0/24`)
- DNS (Do53 + DoT + DoH)
- NTP
- HTTP/HTTPS outbound
- SSH uniquement depuis la workstation d’administration vers l’IP management
- Blocage du trafic vers les réseaux privés (anti-pivot)
- **Block All** en dernière règle

### OPT2 – Zone isolée / Lab (`10.0.20.0/24`)
- DNS + NTP + HTTP/HTTPS outbound
- SSH depuis la workstation d’administration uniquement (optionnel)
- Blocage fort vers les réseaux privés
- **Block All** en dernière règle (isolation forte)

---

## Voir aussi

- [04 – Firewall Rules Hardening & Cleanup](./04%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md) → Détail complet des règles
- [04b – VLANs et Segmentation Réseau](./04b%20-%20VLANs%20et%20Segmentation%20Réseau.md)
- [06 – Aliases OPNsense](./06%20-%20Aliases%20OPNsense.md)
- [09 – Suricata (Intrusion Detection) sur OPNsense](./09%20-%20Suricata%20(Intrusion%20Detection)%20sur%20OPNsense.md)
- [01 – Architecture & Sommaire](./01-architecture.md) (état global actualisé)

---

**Document historique – Snapshot juillet 2026**  
*Version portfolio sanitisée – 15 août 2026*
