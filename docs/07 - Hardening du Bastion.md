

## Philosophie générale
- Firewall sécurisé mais utilisable au quotidien
- Niveau "pro-sérieux" pour homelab / usage personnel
- Défense en profondeur sans se tirer une balle dans le pied

## Configuration matérielle
- SSD 120 Go → OPNsense bare metal
- HDD 1 To → Stockage + coffre-fort
- Carte réseau Intel I226-T4 (4 ports 2.5G)

## Hardening OPNsense (niveaux)

**Niveau 1 – Bases indispensables**
- Changement immédiat mot de passe root
- Activation 2FA sur web + SSH
- Accès web uniquement via WireGuard (pas de WAN direct)
- Fail2ban activé
- SSH uniquement en clé publique

**Niveau 2 – Durcissement avancé**
- PAM durci
- Limitation des tentatives de connexion
- Logs renforcés + rotation
- Suricata en mode IDS (ou IPS)
- Désactivation des services inutiles

**Niveau 3 – Accès distant**
- WireGuard comme seule porte d’entrée
- ACL strictes sur WireGuard
- Pas de Tailscale (contrôle total)

## Coffre-fort sur HDD 1 To
- Partitionnement :
  - `/archive` → stockage non chiffré
  - `/coffre` → conteneur(s) VeraCrypt
- Accès via Samba uniquement depuis LAN ou WireGuard

## Sauvegardes
- Sauvegarde régulière de la config OPNsense
- Sauvegarde du coffre-fort sur support externe ou cloud chiffré

**Dernière mise à jour** : 14 mai 2026




