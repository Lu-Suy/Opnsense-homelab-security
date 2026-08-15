# 14 – Inventaire Hardware & Ressources des machines

**Date originale :** 30 juillet 2026  
**Version portfolio :** 15 août 2026 (sanitisée + enrichie)  
**Objectif :** Vue d’ensemble claire des machines présentes sur le réseau, de leurs ressources hardware, et de leurs capacités / limites opérationnelles.

---

## Pourquoi un inventaire hardware & ressources ?

Dans une architecture de défense, connaître précisément le matériel n’est pas un luxe : c’est une **condition de pilotage**.

**Raisons principales :**

1. **Dimensionnement réaliste**  
   Avant d’ajouter un service (Wazuh, WireGuard, honeypot, laboratoire IA…), on doit savoir si la machine cible peut l’encaisser sans dégrader les services critiques.

2. **Anticipation des points de rupture**  
   Savoir à partir de quelle charge (nombre de conteneurs, volume de logs, nombre d’agents, débit réseau…) les ressources deviennent insuffisantes.

3. **Décisions de migration / upgrade**  
   L’inventaire a permis de décider le passage du bastion de la Prodesk (i3) vers le G4 (i5 8ᵉ gen) une fois les besoins Wazuh + Docker + cloudflared consolidés.

4. **Traçabilité et documentation pro**  
   Un portfolio ou un entretien technique demande souvent « quelles machines, avec quelles capacités, pour quels rôles ».

Sans inventaire à jour, on pilote à l’aveugle.

---

## 1. Tableau récapitulatif global (snapshot 30 juillet 2026)

| Machine / Appareil                    | Rôle principal                              | Zone / IP(s)                                      | CPU / SoC                              | RAM                            | Stockage                              | OS principal               |
|---------------------------------------|---------------------------------------------|---------------------------------------------------|----------------------------------------|--------------------------------|---------------------------------------|----------------------------|
| **Prodesk 600 Mini G2**               | Bastion (BunkerWeb, Docker, cloudflared)    | OPT1 : `10.0.10.x` (services + SSH)              | Intel Core i3-6100T @ 3.20 GHz        | 16 Go                         | 1 To NVMe SSD                        | Debian 12 Bookworm        |
| **HP EliteDesk 800 G2 SFF**           | Firewall OPNsense + Suricata                | WAN / LAN / OPT1 / OPT2                           | Intel Core i5-6500 (4C/4T)            | 16 Go                         | SSD 128 Go + HDD 1 To                | OPNsense (+ Win11 sur HDD)|
| **Workstation d’administration**      | Machine cliente de confiance                | LAN : `10.0.0.x`                                  | AMD Ryzen 7 4800H                     | 64 Go                         | NVMe 500 Go + NVMe 1 To              | Windows                   |
| **ASUS Ascent GX10**                  | Zone isolée / Lab AI                        | OPT2 : `10.0.20.x`                                | NVIDIA GB10 (20 cœurs Arm)            | 128 Go LPDDR5x unifiée        | 1 To M.2 NVMe                        | NVIDIA DGX OS             |
| **Smartphones**                       | Mobilité / tests                            | Wi-Fi / mobile                                    | —                                     | 4–12 Go                       | 64–512 Go                            | Android                   |

> **Note d’évolution (août 2026)** : Le rôle de bastion principal a ensuite été basculé vers un **EliteDesk 800 G4** (i5 8ᵉ gen) sous Rocky Linux 10. La Prodesk reste dans l’inventaire comme machine historique / possible honeypot.

---

## 2. Analyse de capacité et points de rupture

### 2.1 Prodesk 600 Mini G2 (ancien bastion – i3-6100T, 16 Go RAM)

| Ressource | Capacité réelle | Usage typique (juillet 2026) | Point de rupture approximatif |
|-----------|-----------------|------------------------------|-------------------------------|
| **CPU** (2C/4T, 3.2 GHz) | Faible/moyen | BunkerWeb + cloudflared + quelques conteneurs | 4–6 conteneurs actifs + forte charge web → saturation |
| **RAM** 16 Go | Correcte pour un mini-PC 2016 | ~4–6 Go utilisés en nominal | Wazuh all-in-one + plusieurs agents + Docker lourds → risque d’OOM |
| **Stockage** 1 To NVMe | Large | Logs + images Docker | Plusieurs mois de logs détaillés + snapshots → surveillance nécessaire |

**Exemples concrets de limites :**
- **OK** : BunkerWeb multisite léger + Cloudflare Tunnel + Fail2ban
- **Limite** : Ajout de Wazuh (indexer + dashboard + manager) en all-in-one → CPU et RAM deviennent rapidement le goulot
- **Rupture** : 8+ conteneurs + indexation de logs importants + agents Wazuh multiples → machine inconfortable / instable

C’est précisément cette limite qui a motivé la migration vers le G4.

### 2.2 EliteDesk 800 G2 (OPNsense – i5-6500, 16 Go RAM)

| Ressource | Capacité | Commentaire |
|-----------|----------|-----------|
| **CPU** i5-6500 (4C/4T) | Bon pour firewall + IDS | Suricata en mode IDS + règles ET Open reste confortable |
| **RAM** 16 Go | Largement suffisante | OPNsense + Suricata + NetFlow consomment peu |
| **Stockage** SSD 128 Go + HDD 1 To | SSD pour le système, HDD pour Win11 / archives | Attention à la place des logs Suricata / packet capture |

**Point de rupture :**  
Très élevé pour le rôle firewall. Le goulot serait plutôt le **débit réseau** ou le nombre de règles/states que le nombre de machines derrière.  
Avec 4–5 zones (LAN, OPT1, OPT2, VLANs) et Suricata, la machine reste à l’aise.

### 2.3 Workstation d’administration (64 Go RAM)

Machine très confortable. Aucun point de rupture pertinent dans le contexte du lab actuel (administration, tests, navigation, éventuellement agents Wazuh).

### 2.4 ASUS Ascent GX10 (Lab AI)

Conçue pour l’IA (128 Go unifiée + GPU Blackwell).  
Hors scope du bastion classique. Sa limite serait plutôt liée aux modèles LLM / charge GPU que au réseau de défense.

---

## 3. Détail par machine (sanitisé)

### 3.1 Prodesk 600 Mini G2 (Bastion historique)

| Élément              | Valeur                                              |
|----------------------|-----------------------------------------------------|
| **Modèle**           | HP ProDesk 600 Mini G2                              |
| **CPU**              | Intel Core i3-6100T @ 3.20 GHz (2 cœurs / 4 threads)|
| **RAM**              | 16 Go                                               |
| **Stockage**         | SSD NVMe 1 To                                       |
| **OS**               | Debian 12 Bookworm                                  |
| **Rôle**             | BunkerWeb, Docker, cloudflared, services web        |
| **Zone**             | OPT1 (`10.0.10.x`)                                  |

**Notes :**  
Machine suffisante pour un reverse-proxy + tunnel + services légers.  
Insuffisante pour un SIEM all-in-one + charge Docker significative → migration vers G4 justifiée.

### 3.2 HP EliteDesk 800 G2 SFF (Firewall OPNsense)

| Élément              | Valeur                                              |
|----------------------|-----------------------------------------------------|
| **Modèle**           | HP EliteDesk 800 G2 Small Form Factor               |
| **CPU**              | Intel Core i5-6500 (4 cœurs / 4 threads)            |
| **RAM**              | 16 Go                                               |
| **Stockage**         | SSD 128 Go + HDD 1 To (Windows 11 bootable)         |
| **OS principal**     | OPNsense                                            |
| **Rôle**             | Firewall, routing, Suricata IDS, VLANs, NetFlow     |
| **Interfaces**       | WAN, LAN, OPT1 (Bastion), OPT2 (Lab)                |

### 3.3 Workstation d’administration

| Élément           | Valeur                                   |
| ----------------- | ---------------------------------------- |
| **CPU**           | AMD Ryzen 7 4800H (8 cœurs / 16 threads) |
| **RAM**           | 64 Go                                    |
| **Stockage**      | NVMe 500 Go + NVMe 1 To                  |
| **OS**            | Windows                                  |
| **Rôle**          | Machine cliente de confiance / administration |

### 3.4 ASUS Ascent GX10 (Lab AI)

| Élément              | Valeur                                              |
|----------------------|-----------------------------------------------------|
| **SoC**              | NVIDIA GB10 Grace Blackwell                         |
| **RAM**              | 128 Go LPDDR5x unifiée                              |
| **Stockage**         | 1 To M.2 NVMe                                       |
| **Rôle**             | Zone isolée / Lab AI (OPT2)                         |

---

## 4. Synthèse – Capacités vs besoins

| Besoin futur                    | Machine adaptée ?          | Commentaire |
|--------------------------------|----------------------------|-----------|
| BunkerWeb + Tunnel             | Prodesk / G4               | OK |
| Wazuh all-in-one               | Prodesk (limite) → **G4**  | Migration justifiée |
| WireGuard (peu d’utilisateurs) | OPNsense ou G4             | OK |
| Honeypot léger                 | Ancienne Prodesk           | Possible plus tard |
| Lab AI / modèles lourds        | GX10                       | Dédié |

---

## 5. Recommandations

1. Maintenir cet inventaire à jour à chaque changement de rôle ou d’upgrade.
2. Avant d’ajouter un service lourd, vérifier la marge CPU/RAM restante.
3. Documenter les points de rupture observés (c’est de la connaissance opérationnelle).
4. Lier ce document depuis `01-architecture.md`.

---

**Document de référence – Inventaire Hardware & Capacités**  
*Version portfolio sanitisée et enrichie – 15 août 2026*
