# 29 – Fail2ban : protection brute-force SSH (G4)

**Projet :** Horus AIS – bastion  
**Date :** 11 août 2026 (fusion portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Objectif :** Protection active contre les tentatives d’authentification SSH répétées — couche de défense en profondeur, dans une logique d’exploitation (bastion / jump host / data center), pas uniquement « confort homelab ».

> **Publication :** libellés projet sanitisés. Pas d’IP management sensible dans cette fiche.

---

## 1. Contexte et positionnement

Le bastion expose un service SSH **déjà durci** (auth par clé uniquement, utilisateurs restreints, root limité — doc 28).

Même avec un SSH correct, les tentatives restent visibles dans les logs. Sans réaction automatisée, un attaquant peut :

- multiplier les essais de clés / comptes ;
- générer du bruit dans les logs ;
- sonder la surface d’attaque dans la durée.

**Fail2ban** : détection d’échecs répétés → **bannissement temporaire** de l’IP source.

Ce n’est **pas** un substitut au durcissement SSH.  
C’est une couche **defense-in-depth**, standard en exploitation.

---

## 2. Logique professionnelle

| Principe | Application ici |
|----------|-----------------|
| Réduction de surface | SSH déjà durci (doc 28) |
| Détection + réaction | Fail2ban sur les échecs d’auth |
| Traçabilité | journald + compteurs Fail2ban |
| Réversibilité | Unban manuel possible |
| Maintenabilité | Config dans `jail.local` (non écrasée aux updates) |

En environnement data center / entreprise, ce type de contrôle est attendu sur les bastions d’administration et jump hosts — même en réseau interne segmenté.

---

## 3. Installation

```bash
sudo dnf install -y fail2ban fail2ban-firewalld
```

| Paquet | Rôle |
|--------|------|
| `fail2ban` | Moteur de détection et de ban |
| `fail2ban-firewalld` | Actions de ban via firewalld (cohérent avec Rocky) |

```bash
rpm -q fail2ban
# exemple lab : fail2ban-1.1.0-6.el10_0.noarch

systemctl status fail2ban --no-pager
```

Capture lab – paquet installé, service **pas encore** démarré :

![Fail2ban installé – statut avant enable](../images/Pasted%20image%2020260811083800.png)

---

## 4. Configuration retenue

On **ne** modifie **pas** `jail.conf` (fichier du paquet, écrasé aux mises à jour).  
On crée `/etc/fail2ban/jail.local` :

```bash
sudo tee /etc/fail2ban/jail.local > /dev/null << 'EOF'
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3
backend  = systemd
banaction = firewallcmd-rich-rules
banaction_allports = firewallcmd-rich-rules

[sshd]
enabled  = true
port     = ssh
filter   = sshd
maxretry = 3
bantime  = 2h
findtime = 15m
EOF
```

### Paramètres et justification

| Paramètre | Valeur | Justification |
|-----------|--------|---------------|
| `maxretry` | 3 | Aligné avec `MaxAuthTries 3` (doc 28) |
| `findtime` | 10–15 min | Fenêtre de corrélation des échecs |
| `bantime` | 1h (2h pour SSH) | Dissuasif, sans bloquer trop longtemps un admin légitime en erreur |
| `backend = systemd` | journald | Source de logs native et fiable sur Rocky |
| Jail `sshd` seule (pour l’instant) | — | Priorité au service d’accès le plus sensible |

> Note lab : selon versions / docs, `logpath = /var/log/secure` peut apparaître ; avec `backend = systemd`, Fail2ban s’appuie sur journald — c’est la voie retenue ici.

---

## 5. Activation

```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban --no-pager
```

Capture lab – service **active (running)** :

![Fail2ban active (running) après enable --now](../images/Pasted%20image%2020260811084046.png)

---

## 6. Vérification des jails

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

- Première commande : liste des jails actives  
- Seconde : détail jail SSH (échecs, total, IP bannies)

Capture lab – jail `sshd` active, 0 IP bannie au démarrage (normal) :

![fail2ban-client status sshd – jail active](../images/Pasted%20image%2020260811084337.png)

| Élément | Statut lab |
|---------|------------|
| Fail2ban | Running |
| Jail `sshd` | Active |
| Backend | journald |
| IP bannies au premier check | 0 (attendu) |

---

## 7. Exploitation courante

### Consulter l’état

```bash
sudo fail2ban-client status sshd
```

### Suivre l’activité

```bash
sudo tail -f /var/log/fail2ban.log
# ou
journalctl -u fail2ban -f
```

### Débannir une IP (procédure d’exploitation)

```bash
sudo fail2ban-client set sshd unbanip <ADRESSE_IP>
```

Cas typique : administrateur s’est trompé de clé / de compte et s’est fait bannir.

---

## 8. Ce que cette mise en place démontre (angle pro / entretien)

| Compétence | Manifestation concrète |
|------------|------------------------|
| Durcissement d’accès distant | SSH + Fail2ban cohérents |
| Defense-in-depth | Auth forte + réaction automatique |
| Exploitation | Statut, unban, logs |
| Bonnes pratiques de config | `jail.local`, pas de modification des fichiers paquet |
| Alignement paramètres | `maxretry` Fail2ban ↔ `MaxAuthTries` SSH |

En entretien, on peut expliquer :

1. pourquoi Fail2ban ne remplace pas un SSH correctement configuré ;  
2. comment les seuils sont alignés avec le durcissement existant ;  
3. comment on opère un ban / unban en conditions réelles.

---

## 9. Limites et perspectives

Fail2ban est une **protection locale réactive**. En environnement d’envergure, on complète typiquement par :

| Couche | Rôle |
|--------|------|
| Segmentation réseau / firewall amont | Qui peut joindre le port 22 |
| VPN / WireGuard (accès admin) | Ne plus exposer SSH « nu » |
| SIEM / Wazuh | Corrélation centralisée, alertes, historique |
| Bastion dédié + MFA | Contrôle d’accès renforcé |
| Revue des clés et des comptes | Gouvernance des accès |

Sur ce bastion, les prochaines briques naturelles : **Wazuh**, **WireGuard**, durcissement applicatif BunkerWeb (mode block / ModSecurity).

---

## 10. État final – 11 août 2026

| Élément | Statut |
|---------|--------|
| Fail2ban installé | OK (1.1.0 lab) |
| Service enabled + running | OK |
| Jail `sshd` active | OK |
| Backend journald | OK |
| Config dans `jail.local` | OK |
| Alignement avec durcissement SSH (doc 28) | OK |

---

## 11. Commandes de contrôle rapide

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
journalctl -u fail2ban -n 50 --no-pager
sudo fail2ban-client set sshd unbanip <IP>
```

---

**Document fusionné – version portfolio / runbook – 15 août 2026**  
**Orientation :** exploitation / data center / défense en profondeur.
