# 09 – Suricata (Intrusion Detection) sur OPNsense

**Date originale** : 5 juillet 2026  
**Dernière mise à jour** : 5 juillet 2026  
**Version OPNsense** : 26.1.11_6  
**Version Suricata** : 8.0.5  
**Statut** : ✅ Installé et configuré en mode IDS passif  
**Version portfolio** : 15 août 2026 (enrichie + sanitisée)

> **Note portfolio** :  
> Document enrichi pour expliquer le *pourquoi* des choix techniques.  
> Les noms d’interfaces et de policy ont été généralisés.  
> **Toutes les captures d’écran et le contenu technique original sont conservés.**

---

## Philosophie : pourquoi Suricata dans cette architecture ?

Dans une architecture de défense en profondeur :
- Le **firewall (OPNsense)** filtre selon des règles explicites (default-deny).
- **Suricata** apporte la couche de **détection d’intrusion** : il analyse le trafic en profondeur (signatures, anomalies, protocoles) et génère des alertes.

Même un firewall bien configuré ne voit pas tout. Un attaquant peut utiliser des ports légitimes ou exploiter des faiblesses applicatives. Suricata détecte ces comportements.

On commence volontairement en mode **détection pure (IDS)** avant d’envisager le mode prévention (IPS).

---

## Installation des packages sur OPNsense (procédure générale)

1. Aller dans **System → Firmware → Packages**  
2. Cliquer sur l’onglet **Available packages** (paquets disponibles)  
3. Utiliser la barre de recherche pour trouver le package voulu (ex. : `suricata`)  
4. Cliquer sur le bouton **Install** à droite du package  
5. Attendre la fin de l’installation (le bouton passe à **Reinstall** une fois terminé)

**Vérification** :  
- Retourner dans **System → Firmware → Packages**  
- Le package doit apparaître avec le statut **Installed** ou le bouton **Reinstall**

## Installation de Suricata

- Package à installer : **`os-suricata`**  
- Une fois installé, Suricata apparaît dans le menu **Services → Intrusion Detection**

## Configuration initiale de Suricata

1. Aller dans **Services → Intrusion Detection**  
2. Onglet **Administration** (ou Settings)  
3. Cocher **Enabled**  
4. **Capture mode** : `PCAP live mode (IDS)` (mode détection seulement, recommandé au début)  
5. **Interfaces** : Sélectionner `WAN`, `LAN`, `OPT1`, `OPT2` (et `OPT3` si utilisé)  
6. Cliquer sur **Apply**

### Pourquoi le mode IDS (PCAP live mode) d’abord ?

- Éviter de bloquer du trafic légitime par des faux positifs dès le premier jour.
- Observer et comprendre le trafic réel de l’infrastructure.
- Affiner les règles et les seuils avant de passer en mode « Drop » (IPS).
- Démarche professionnelle : **détecter → comprendre → affiner → bloquer**.

### Pourquoi surveiller toutes les interfaces ?

- **WAN** : trafic entrant depuis Internet  
- **LAN** : détection de pivots / mouvements latéraux  
- **OPT1 (Bastion)** : zone des services exposés  
- **OPT2 (Lab)** : zone isolée  

Surveiller uniquement le WAN laisserait passer les menaces internes.

## Téléchargement des règles

1. Aller dans l’onglet **Download**  
2. Sélectionner uniquement les rulesets recommandés (voir ci-dessous)  
3. Cliquer sur **Download & Update Rules**

**Note** : Ne pas tout cocher pour éviter trop de faux positifs et de charge inutile.

### Pourquoi ces rulesets (et pas tous) ?

On active un ensemble ciblé (ET Open + abuse.ch) pour :
- Couvrir les menaces courantes (malware, exploit, phishing, scan, botnets…)
- Limiter les faux positifs et la charge CPU
- Garder des alertes lisibles et actionnables

---

## Configuration actuelle (5 juillet 2026)

**Mode** : IDS (PCAP live mode) – Détection + logs uniquement

**Interfaces surveillées** : WAN, LAN, OPT1 (Bastion), OPT2 (Lab)

**Logging** :
- EVE syslog output : ✅ activé (préparation Wazuh)
- Logs rotation : Daily, conservation 7 jours

**Observations** :
- Aucune alerte pour le moment (normal au premier jour).
- Warnings flowbit dans les logs : classiques et non bloquants.
- Suricata tourne correctement.

**Rulesets activés** :
- ET open/botcc
- ET open/botcc.portgrouped
- ET open/compromised
- ET open/emerging-exploit
- ET open/emerging-malware
- ET open/emerging-phishing
- ET open/emerging-policy
- ET open/emerging-scan
- ET open/emerging-web_client
- ET open/emerging-web_server
- abuse.ch/Feodo Tracker
- abuse.ch/SSL Fingerprint Blacklist
- abuse.ch/SSL IP Blacklist
- abuse.ch/ThreatFox

**Mode** : PCAP live mode (IDS)

---

## 1. Configuration globale

- **Policy créée** : `Bastion-Policy`
- **Mode** : PCAP live mode (IDS)
- **Action par défaut** : Alert
- **Interfaces activées** : WAN, LAN, OPT1, OPT2 (toutes les interfaces)

*(Insère ici capture d’écran de la policy Bastion-Policy)*

## 2. Administration → General Settings

- **Enabled** : ✅
- **Capture mode** : PCAP live mode (IDS)
- **Interfaces** : LAN, OPT1, OPT2, WAN (Select All)
- **Promiscuous mode** : décoché (pas nécessaire)

## 3. Logging (EVE JSON)

- **Enable eve syslog output** : ✅ coché
- Fichier généré : `/var/log/suricata/eve.json` (format JSON structuré pour future intégration Wazuh)

**Pourquoi EVE JSON / syslog ?**
- Format structuré moderne, facilement ingérable par un SIEM.
- Préparation directe de l’intégration Suricata → Wazuh.
- Rotation quotidienne + conservation 7 jours pour ne pas saturer le disque.

![Capture d’écran](../images/Pasted%20image%2020260705203847.png)

## 4. Log File (état actuel au premier démarrage)

- Warnings classiques « flowbit … is checked but not set » → **normal** au premier lancement (initialisation des règles ET Open).
- Recherche `alert` ou `eve` dans Log File : encore vide → normal (pas encore de trafic malveillant détecté).

![Capture d’écran](../images/Pasted%20image%2020260705210504.png)

## 5. Prochaines étapes

- Observer les logs pendant quelques jours (trafic normal + tests d’attaque simulés).
- Affiner les règles (supprimer les faux positifs).
- Passer en mode IPS (changer Action en « Drop ») une fois stable.
- Intégration Suricata → Wazuh via EVE JSON.

---

### 6. Documentation Monitoring & NetFlow

# Configuration NetFlow & Insight OPNsense

**Date** : 05/07/2026  
**Version OPNsense** : 26.1.8_5

## Configuration appliquée
- Listening interfaces : OPT1, LAN, WAN
- Capture local : Yes
- Version : v9
- Destinations : vide (géré par Capture local)
- Timeouts : valeurs par défaut

![Capture d’écran](../images/Pasted%20image%2020260705211536.png)

## Vérification
```bash
# Sur console OPNsense
ps aux | grep flow
```

**Objectif** : Compléter Suricata avec visibilité flux (top talkers, anomalies, etc.).

### Pourquoi NetFlow en complément de Suricata ?

- **Suricata** = détection de signatures et comportements suspects  
- **NetFlow** = vision quantitative des flux (qui parle à qui, volumes, top talkers)

Les deux se complètent parfaitement.

---

**Document de référence – Suricata IDS sur OPNsense**  
*Version portfolio enrichie – 15 août 2026*
