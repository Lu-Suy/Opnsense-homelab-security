# 07 – Hardening du Bastion

**Date originale** : 14 mai 2026  
**Version portfolio** : 15 août 2026 (enrichie + sanitisée)  
**Objectif** : Définir la philosophie et les niveaux de durcissement appliqués à l’infrastructure de défense (OPNsense + bastion).

> **Note portfolio** : Document de référence sur la posture de sécurité. Les éléments d’identification ont été généralisés.

---

## 1. Philosophie générale

L’approche de hardening repose sur trois principes :

- **Défense en profondeur** : plusieurs couches de protection (réseau, système, accès, monitoring) plutôt qu’une seule barrière.
- **Utilisable au quotidien** : le durcissement ne doit pas rendre l’administration impossible ou trop pénible.
- **Niveau professionnel pour un contexte lab / personnel** : on vise un standard « pro-sérieux » sans tomber dans le parano excessif qui casse l’usage.

Le but n’est pas d’être invulnérable (personne ne l’est), mais de **réduire fortement la surface d’attaque**, de **détecter** les tentatives et de **limiter les dégâts** en cas de compromission.

---

## 2. Pourquoi ces choix de hardening ?

| Domaine | Choix principal | Pourquoi ? |
|---------|-----------------|----------|
| Accès distant | WireGuard (ou tunnel Cloudflare) plutôt que ports ouverts | Réduit massivement l’exposition. Un port 22/443 ouvert sur Internet attire les scans 24/7. |
| Authentification | Clés SSH uniquement + (idéalement) 2FA | Les mots de passe sont faibles face aux attaques par force brute / credential stuffing. |
| Utilisateurs | Séparation « utilisateur limité » / « admin full » | Principe du moindre privilège. On ne travaille pas en root ou en admin permanent. |
| Fail2ban / rate limiting | Ban automatique après quelques échecs | Bloque les botnets et scripts de brute-force avant qu’ils ne deviennent un problème. |
| Logs + monitoring | Logs activés partout + SIEM (Wazuh) | Sans visibilité, on ne détecte rien. Les logs sont la base de la détection. |
| Segmentation | Zones LAN / OPT1 / OPT2 + default-deny | Limite les pivots. Une machine compromise ne doit pas pouvoir tout atteindre. |

---

## 3. Configuration matérielle (Machine OPNsense – HP EliteDesk)

- **SSD 120 Go** → OPNsense en bare metal
- **HDD 1 To** → Stockage + coffre-fort (Windows 11 installé dessus)
- **Carte réseau** : Intel I226-T4 (4 ports 2.5G)

Cette machine (EliteDesk) est le pare-feu / routeur de l’infrastructure.  
Le HDD 1 To sert notamment de support pour le coffre-fort chiffré et le stockage.

---

## 4. Niveaux de hardening (référence)

### Niveau 1 – Bases indispensables

À faire **immédiatement** sur toute machine exposée ou critique :

- Changement immédiat des mots de passe par défaut
- Activation de l’authentification à deux facteurs (2FA) sur les interfaces d’administration
- Accès web d’administration **jamais** directement depuis Internet (uniquement via VPN ou tunnel)
- Fail2ban (ou équivalent) activé
- SSH en clé publique uniquement (PasswordAuthentication no)

### Niveau 2 – Durcissement avancé

- Durcissement PAM / limitation des tentatives
- Logs renforcés + rotation correcte
- Suricata (IDS/IPS) sur OPNsense
- Désactivation de tous les services inutiles
- Séparation stricte des utilisateurs (least privilege)

### Niveau 3 – Accès distant maîtrisé

- WireGuard (ou Cloudflare Tunnel) comme **seule** porte d’entrée distante
- ACL strictes sur le VPN (seules les machines de confiance)
- Pas de solution « zero-config » type Tailscale si on veut garder le contrôle total de l’infrastructure

---

## 5. Hardening du bastion (machine)

Sur le bastion lui-même (ancien Prodesk / actuel G4), les points clés appliqués sont :

- **Séparation des utilisateurs** : un compte limité pour le quotidien + un compte admin dédié
- **SSH durci** : clés uniquement, AllowUsers restreint, MaxAuthTries bas, root login interdit
- **Fail2ban** actif sur SSH
- **Mises à jour** régulières et patchs de sécurité
- **Docker** et services exposés uniquement sur l’IP services (séparation plan management / plan services)

Le détail pratique de ces configurations se trouve dans les documents 07b, 28 et 29 (G4).

---

## 6. Coffre-fort / stockage sensible

Sur un disque secondaire :
- Partition non chiffrée pour archives
- Conteneur(s) chiffré(s) (VeraCrypt ou LUKS) pour les données sensibles
- Accès limité (Samba uniquement depuis LAN ou VPN)

Cette partie peut être reprise plus tard selon les besoins.

---

## 7. Sauvegardes

- Sauvegarde régulière de la configuration OPNsense
- Sauvegarde des données critiques (config bastion, volumes Docker importants, etc.)
- Support externe ou cloud chiffré recommandé

---

## 8. Voir aussi

- [07b – Hardening du Bastion (détail pratique)](./07b%20-%20Hardening%20du%20Bastion%20Godmode.md)
- [28 – Durcissement SSH (G4)](./28%20-%20Durcissement%20SSH%20(G4).md)
- [29 – Fail2ban protection brute-force SSH (G4)](./29%20-%20Fail2ban%20protection%20brute-force%20SSH%20(G4).md)
- [04 – Firewall Rules Hardening & Cleanup](./04%20-%20Firewall%20Rules%20Hardening%20%26%20Cleanup.md)

---

**Document de référence – Philosophie et niveaux de hardening**  
*Version portfolio enrichie – 15 août 2026*
