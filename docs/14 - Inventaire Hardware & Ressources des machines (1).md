# 14 – Inventaire Hardware & Ressources des machines

**Projet :** Bastion Godmode / OPNsense Homelab Security  
**Date :** 30 juillet 2026  
**Dernière mise à jour :** 30 juillet 2026 (CPU Prodesk + EliteDesk confirmés, SSD AlphaDeck, IPs complètes)  
**Objectif :** Vue d’ensemble claire et à jour de toutes les machines et appareils présents sur le réseau, avec leurs ressources hardware (base usine + upgrades) et adresses IP.

---

## 1. Tableau récapitulatif global

| Machine / Appareil                    | Rôle principal                              | Zone / IP(s)                                      | CPU / SoC                              | RAM                            | Stockage                              | OS principal               | Upgrades notables                                      |
|---------------------------------------|---------------------------------------------|---------------------------------------------------|----------------------------------------|--------------------------------|---------------------------------------|----------------------------|--------------------------------------------------------|
| **Prodesk 600 Mini G2**               | Bastion principal (BunkerWeb, Fail2ban, Docker) | OPT1 : **10.0.10.10** (services)<br>**10.0.10.11** (SSH) | **Intel Core i3-6100T @ 3.20 GHz**    | 16 Go                         | 1 To NVMe SSD                        | Debian 12 Bookworm        | —                                                      |
| **HP EliteDesk 800 G2 SFF**           | Firewall OPNsense + Suricata                | WAN : **192.168.5.244**<br>LAN : **10.0.0.1**<br>OPT1 : **10.0.10.1**<br>OPT2 : **10.0.20.1** | **Intel Core i5-6500** (4C/4T)        | **16 Go** (2×8 Go 3200 MHz)   | **SSD 128 Go** + **HDD 1 To**        | OPNsense (+ Win11 sur HDD)| RAM 16 Go, SSD 128 Go, HDD 1 To (Win11 bootable)       |
| **MSI Alpha 15 A4DEK (AlphaDeck)**    | Machine cliente de confiance                | LAN : **10.0.0.10**<br>IP de secours : *à définir* | AMD Ryzen 7 4800H                     | **64 Go** (2×32 Go 3200 MHz)  | **NVMe 500 Go** + **NVMe 1 To**      | Windows                   | RAM → 64 Go                                            |
| **ASUS Ascent GX10**                  | Zone isolée / Lab AI                        | OPT2 : **10.0.20.x**                              | NVIDIA GB10 (20 cœurs Arm)            | 128 Go LPDDR5x unifiée        | **1 To** M.2 NVMe                    | NVIDIA DGX OS             | Aucun (stockage upgradable plus tard)                  |
| **Samsung Galaxy Note20 Ultra**       | Smartphone principal                        | Wi-Fi / mobile                                    | Exynos / Snapdragon                   | 8/12 Go                       | 256/512 Go                           | Android                   | —                                                      |
| **Samsung Galaxy A50**                | Smartphone secondaire                       | Wi-Fi / mobile                                    | Exynos 9610                           | 4/6 Go                        | 64/128 Go                            | Android                   | —                                                      |
| **Sony Xperia Z1**                    | Micro + caméra éventuelle pour PC           | —                                                 | Qualcomm Snapdragon 800               | 2 Go                          | 16 Go (+ microSD)                    | Android (ancien)          | Usage secondaire (mic/caméra)                          |

---

## 2. Détail par machine

### 2.1 Prodesk 600 Mini G2 (Bastion principal)

| Élément              | Valeur                                              |
|----------------------|-----------------------------------------------------|
| **Modèle**           | HP ProDesk 600 Mini G2 / 600 G2 DM                  |
| **Hostname**         | bastion-godmode                                     |
| **CPU**              | **Intel Core i3-6100T @ 3.20 GHz** (2 cœurs / 4 threads) |
| **RAM**              | 16 Go                                               |
| **Stockage**         | SSD NVMe 1 To                                       |
| **OS**               | Debian 12 Bookworm                                  |
| **Rôle**             | BunkerWeb, Fail2ban, Docker, services web, SSH durci |
| **IP Services**      | **10.0.10.10/24**                                   |
| **IP Management/SSH**| **10.0.10.11/24**                                   |
| **Gateway**          | 10.0.10.1 (OPNsense OPT1)                           |
| **Upgrades**         | Aucun upgrade hardware majeur documenté             |
| **Document lié**     | `05 - État de la machine Prodesk`                   |

**Notes :**  
CPU confirmé le 30 juillet 2026 via `lscpu`. Machine suffisante pour BunkerWeb + Cloudflare Tunnel + services légers.

---

### 2.2 HP EliteDesk 800 G2 SFF (Firewall OPNsense)

| Élément              | Valeur                                              |
|----------------------|-----------------------------------------------------|
| **Modèle**           | HP EliteDesk 800 G2 Small Form Factor               |
| **CPU**              | **Intel Core i5-6500** (4 cœurs / 4 threads)        |
| **RAM**              | **16 Go** (2×8 Go 3200 MHz)                         |
| **Stockage**         | **SSD 128 Go** (ajouté) + **HDD 1 To** (ajouté)     |
| **OS principal**     | OPNsense                                            |
| **OS secondaire**    | Windows 11 (sur le HDD 1 To, bootable)              |
| **Rôle**             | Firewall, routing, Suricata IDS, VLANs, NetFlow     |
| **IP WAN**           | **192.168.5.244/24**                                |
| **IP LAN**           | **10.0.0.1/24**                                     |
| **IP OPT1 (Bastion)**| **10.0.10.1/24**                                    |
| **IP OPT2 (GX10)**   | **10.0.20.1/24**                                    |
| **Upgrades réalisés**| • RAM → 16 Go (2×8 Go 3200 MHz)<br>• Ajout SSD 128 Go<br>• Ajout HDD 1 To avec Windows 11 bootable |



---

### 2.3 MSI Alpha 15 A4DEK (AlphaDeck)

| Élément           | Valeur                                   |
| ----------------- | ---------------------------------------- |
| **Modèle**        | MSI Alpha 15 A4DEK (série Alpha 15)      |
| **CPU**           | AMD Ryzen 7 4800H (8 cœurs / 16 threads) |
| **GPU**           | AMD Radeon RX 5600M 6 Go GDDR6           |
| **RAM**           | **64 Go** (2×32 Go DDR4 3200 MHz)        |
| **Stockage**      | **NVMe 500 Go** + **NVMe 1 To**          |
| **Écran**         | 15,6" FHD IPS 144 Hz                     |
| **OS**            | Windows                                  |
| **Rôle**          | Machine cliente de confiance             |
| **IP principale** | **10.0.0.10/24** (LAN)                   |
| **IP de secours** | 192.168.5.1**/24:22:443                  |
| **Upgrades**      | RAM **64 Go** (2×32 Go 3200 MHz)         |

**Notes :**  
Machine d’administration principale. L’IP de secours reste à préciser (éventuelle IP fixe secondaire ou Wi-Fi).

---

### 2.4 ASUS Ascent GX10

| Élément              | Valeur                                              |
|----------------------|-----------------------------------------------------|
| **Modèle**           | ASUS Ascent GX10                                    |
| **SoC**              | NVIDIA GB10 Grace Blackwell Superchip               |
| **CPU**              | 20 cœurs Arm (10× Cortex-X925 + 10× Cortex-A725)    |
| **GPU**              | NVIDIA Blackwell (intégré) – jusqu’à 1 PFLOP AI     |
| **RAM**              | 128 Go LPDDR5x **unifiée** (CPU + GPU)              |
| **Stockage**         | **1 To** M.2 NVMe                                   |
| **OS**               | NVIDIA DGX OS                                       |
| **Réseau**           | 10 GbE + NVIDIA ConnectX-7 + Wi-Fi 7                |
| **Forme**            | Ultra-compact 150 × 150 × 51 mm                     |
| **Rôle**             | Zone isolée / Lab AI                                |
| **Zone IP**          | OPT2 – **10.0.20.x**                                |
| **Upgrades**         | Aucun pour le moment (stockage upgradable plus tard)|

**Notes :**  
Aucune modification hardware. Stockage upgradable vers 2 To ou 4 To quand les prix seront plus abordables.

---

### 2.5 Appareils mobiles

| Appareil                        | Rôle                          | Notes                                      |
|--------------------------------|-------------------------------|--------------------------------------------|
| **Samsung Galaxy Note20 Ultra**| Smartphone principal          | Wi-Fi / 4G-5G                              |
| **Samsung Galaxy A50**         | Smartphone secondaire         | Usage secondaire                           |
| **Sony Xperia Z1**             | Micro + caméra éventuelle     | Ancien smartphone (2013), usage lab        |
#### Samsung Galaxy Note20 Ultra
- Smartphone principal
- RAM / stockage selon version (généralement 8 ou 12 Go RAM, 256/512 Go)
- Usage : mobilité, tests, notifications

#### Samsung Galaxy A50
- Smartphone secondaire
- Exynos 9610, 4/6 Go RAM, 64/128 Go
- Usage secondaire

#### Sony Xperia Z1
- Ancien smartphone (2013)
- Snapdragon 800, 2 Go RAM, 16 Go + microSD
- **Usage envisagé :** micro et/ou caméra pour le PC (via USB / applications)
---

## 3. Adresses IP – Vue d’ensemble

| Machine              | Interface / Rôle       | Adresse IP          |
|----------------------|------------------------|---------------------|
| EliteDesk (OPNsense) | WAN                    | 192.168.5.244/24    |
| EliteDesk (OPNsense) | LAN                    | 10.0.0.1/24         |
| EliteDesk (OPNsense) | OPT1 (Bastion)         | 10.0.10.1/24        |
| EliteDesk (OPNsense) | OPT2 (GX10)            | 10.0.20.1/24        |
| Prodesk              | Services (BunkerWeb)   | **10.0.10.10/24**   |
| Prodesk              | Management / SSH       | **10.0.10.11/24**   |
| AlphaDeck            | LAN principale         | **10.0.0.10/24**    |
| AlphaDeck            | IP de secours          | *À définir*         |
| GX10                 | OPT2                   | 10.0.20.x           |

---

## 4. Observations & recommandations

1. **Prodesk** : CPU confirmé i3-6100T – correct pour un mini PC de 2016, suffisant pour le rôle actuel.
2. **EliteDesk** : CPU i5-6500 confirmé + upgrades RAM/stockage bien documentés.
3. **AlphaDeck** : Très bien équipé (64 Go RAM + double NVMe). L’IP de secours reste à documenter.
4. Mettre à jour ce fichier à chaque changement d’IP ou d’upgrade hardware.
5. Lier ce document depuis `01-architecture.md`.

---

**Dernière mise à jour :** 30 juillet 2026  
**Document prêt pour intégration dans le vault Obsidian / GitHub (`docs/14 - Inventaire Hardware & Ressources des machines.md`)**
