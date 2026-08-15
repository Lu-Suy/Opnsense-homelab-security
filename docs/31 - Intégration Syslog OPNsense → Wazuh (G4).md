# 31 – Intégration Syslog OPNsense → Wazuh (G4)

**Projet :** Horus AIS – bastion  
**Date :** 12 août 2026 (fusion portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Stack :** Wazuh 4.14.7 (all-in-one)  

**Objectif :** centraliser les logs firewall (`filterlog`) et Suricata d’OPNsense dans Wazuh — visibilité immédiate, bon ROI lab / démo, sans agent FreeBSD.

> **Publication :** IP management masquée (`10.0.10.y` = bastion, `10.0.10.1` = OPNsense côté OPT1). Hostname `bastion`.

---

## 1. Architecture du flux

```text
OPNsense (10.0.10.1)
        │
        │  UDP/514 (syslog)
        ▼
G4 / Wazuh Manager (10.0.10.y)
        │
        ├── wazuh-remoted (écoute syslog)
        ├── analysisd + decoders
        └── archives.log + Dashboard (Discover)
```

**Décision figée :** pas d’agent Wazuh sur OPNsense (FreeBSD, support limité). On utilise le **remote syslog** natif.

---

## 2. Prérequis

| Élément | Valeur / état |
|---------|----------------|
| IP OPNsense (OPT1) | `10.0.10.1` |
| IP Wazuh (G4) | `10.0.10.y` |
| Port syslog | UDP **514** |
| firewalld sur G4 | Actif |
| Wazuh | 4.14.7 all-in-one opérationnel (fiches 30 / 30a) |
| Alias recommandés | host bastion + port `Wazuh_Syslog_514` |

Checks rapides sur le G4 avant modification :

```bash
sudo ss -tulnp | grep 514
sudo firewall-cmd --state
sudo grep -A 15 '<remote>' /var/ossec/etc/ossec.conf
```

---

## 3. Configuration côté Wazuh (G4)

### 3.1 Bloc remote syslog

Éditer `/var/ossec/etc/ossec.conf` et ajouter **après** le bloc remote *secure* (agents), **sans** le remplacer :

```xml
<!-- Syslog from OPNsense -->
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.0.10.1</allowed-ips>
  <local_ip>10.0.10.y</local_ip>
</remote>
```

| Directive | Rôle |
|-----------|------|
| `connection>syslog` | Écoute syslog (pas le canal agents) |
| `allowed-ips` | **Uniquement** l’IP OPNsense OPT1 |
| `local_ip` | Bind sur l’IP management du bastion |

![Bloc remote syslog dans ossec.conf](../images/Pasted%20image%2020260812104617.png)

### 3.2 Ouverture firewalld

```bash
sudo firewall-cmd --add-port=514/udp --permanent
sudo firewall-cmd --reload
```

### 3.3 `logall` (debug temporaire)

Dans la section **`<global>` uniquement** :

```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```

> Utile pour valider le flux dans `archives.log`. À repasser à `no` si le volume devient trop élevé.

#### Problème rencontré – `logall` au mauvais endroit

**Symptôme :** `wazuh-manager` refuse de démarrer / reste down après édition.

**Cause lab :**  
`<logall>` / `<logall_json>` placés dans `<logging>`, ou second bloc `<ossec_config>` collé en fin de fichier (fichier mal formé).

**Règle Wazuh :** ces balises n’appartiennent qu’à **`<global>`**.  
Un seul `</ossec_config>` en fin de fichier.

**Correction :** remettre `yes` uniquement dans `<global>`, supprimer les doublons hors `<global>` et le second `ossec_config`, puis :

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

### 3.4 Redémarrage et écoute

```bash
sudo systemctl restart wazuh-manager
sudo ss -tulnp | grep 514
```

Attendu :

```text
udp UNCONN 0 0 10.0.10.y:514 0.0.0.0:* users:(("wazuh-remoted",...))
```

---

## 4. Configuration côté OPNsense

### 4.1 Destination Remote Logging

**System → Settings → Logging → Remote → +**

| Champ | Valeur recommandée |
|-------|--------------------|
| Enabled | ✔ |
| Transport | UDP(4) |
| Hostname / IP | `10.0.10.y` |
| Port | `514` |
| Applications | d’abord **Select All** pour débloquer, puis affiner (firewall + suricata) |
| Levels | info ou notice |
| Facilities | **Select All** (souvent critique) |
| RFC5424 | Décoché |
| Description | Wazuh G4 – Syslog |

**Save + Apply.**

![Remote logging OPNsense](../images/Pasted%20image%2020260812123835.png)

### 4.2 Redémarrage du service de logging

```bash
service syslog-ng restart
# ou selon version : pluginctl -s syslog restart
```

![Restart syslog-ng](../images/Pasted%20image%2020260812124430.png)

---

## 5. Règle firewall OPT1

Les messages **sortent** d’OPNsense vers le bastion → règle sur **OPT1**, direction **Out**.

| Champ | Valeur |
|-------|--------|
| Action | Pass |
| Interface | OPT1 (bastion) |
| Direction | **Out** |
| Protocol | UDP |
| Source | `10.0.10.1` ou This Firewall |
| Destination | alias host bastion |
| Destination Port | alias `Wazuh_Syslog_514` (= 514) |
| Log | ✔ |
| Description | Allow Syslog to Wazuh G4 |

**Position :** relativement haut dans la liste OPT1.

**Alias port :**

| Champ | Valeur |
|-------|--------|
| Name | `Wazuh_Syslog_514` |
| Type | Port(s) |
| Content | `514` |
| Description | Syslog UDP vers Wazuh G4 |

---

## 6. Tests et validation

### 6.1 Test forcé (`logger` FreeBSD)

Sur le shell OPNsense :

```bash
logger -h 10.0.10.y -P 514 "Test syslog from OPNsense to Wazuh G4"
```

Syntaxe FreeBSD : `-h` host, `-P` port (pas l’option Linux `-n`).

![Test logger OPNsense](../images/Pasted%20image%2020260812121040.png)

Sur le G4 :

```bash
sudo grep -i "Test syslog from OPNsense" /var/ossec/logs/archives/archives.log
```

![Message de test dans archives.log](../images/Pasted%20image%2020260812121020.png)

### 6.2 Réseau

```bash
sudo tcpdump -i any -n udp port 514 -vv
```

### 6.3 Logs réels (`filterlog`)

```bash
sudo tail -f /var/ossec/logs/archives/archives.log | grep -iE "filterlog|suricata|OPNsense|10.0.10.1"
```

![filterlog réel dans archives](../images/Pasted%20image%2020260812123739.png)

**Résultat lab (12 août 2026) :** après **Select All** Applications + restart syslog-ng, les lignes `filterlog` (pass + block) arrivent correctement.

![Discover / événements](../images/Pasted%20image%2020260812125717.png)

---

## 7. Points critiques & leçons

| Point | Détail |
|-------|--------|
| Direction règle | **Out** (logs générés par OPNsense) |
| Source | `10.0.10.1` ou This Firewall |
| Facilities | **Select All** souvent nécessaire au premier flux |
| Applications | Large pour débloquer, puis restreindre |
| `logall` | Uniquement dans `<global>` |
| VPN full-tunnel client | Réduit fortement le volume `filterlog` visible |
| Case « Log » sur les règles FW | Sans log, peu de `filterlog` |
| Decoders | Les decoders type pfSense / filterlog de Wazuh suffisent souvent au début |

### Affinage volume (après validation)

Select All était utile pour **prouver** le chemin. Ensuite :

| Champ | Valeur recommandée | Pourquoi |
|-------|--------------------|----------|
| Applications | firewall + suricata (+ system si besoin) | ROI / bruit |
| Levels | info ou notice | Évite le flood debug |
| Facilities | Select All ou laisser | Moins critique une fois le flux OK |

---

## 8. Checklist finale

- [x] Bloc `<remote>` syslog dans `ossec.conf`
- [x] Port 514/udp ouvert dans firewalld
- [x] Écoute confirmée (`ss -tulnp`)
- [x] Destination Remote créée + Applied sur OPNsense
- [x] Règle OPT1 **Out** + UDP + alias port
- [x] Test `logger` réussi
- [x] `filterlog` réel dans `archives.log`
- [x] Visualisation Dashboard (Discover)
- [ ] `logall` repassé à `no` (optionnel, selon volume)

---

## 9. Suite recommandée

1. Fiche **31A** – filtres Discover / alertes OPNsense  
2. Fiche **32** – agent Wazuh sur AlphaDeck (Windows)  
3. Collecte logs BunkerWeb / Docker  
4. Decoders / rules plus spécifiques si besoin  

---

## Références

- Fiche **30** – Installation Wazuh SIEM (G4)  
- Fiche **30a** – Paramétrage initial Wazuh  
- Fiche **04b** – Firewall / segmentation  
- Documentation Wazuh 4.x – remote syslog  
- OPNsense – System → Settings → Logging / Targets  

---

**Document fusionné – version portfolio / runbook – 15 août 2026**  
**Statut lab :** opérationnel (12 août 2026)
