# 31A – Filtres Discover & Alertes intelligentes OPNsense → Wazuh

**Projet :** Horus AIS – bastion  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Date :** 12 août 2026 (portfolio 15 août 2026)  
**Suite de :** 31 – Intégration Syslog OPNsense → Wazuh  

> **Publication :** libellés projet sanitisés. Adresses côté WAN / broadcast représentées de façon générique (`192.168.5.z` = IP publique lab, `.255` = broadcast).

---

## 1. Objectif

Après avoir centralisé les logs firewall d’OPNsense dans Wazuh (fiche 31), cette fiche vise à :

- créer des **filtres Discover** utiles et durables ;
- comprendre les logs réellement reçus ;
- préparer des **alertes intelligentes** démontrables (employeur / jury) ;
- identifier le besoin de simulations d’attaque **contrôlées**.

---

## 2. État constaté (lab)

### 2.1 Ce qui fonctionne

- Les logs `filterlog` d’OPNsense arrivent bien dans Wazuh  
- La règle de corrélation **87702** (*Multiple pfSense firewall blocks from same source*) se déclenche (level 10)  
- Les champs structurés existent (`data.srcip`, `data.dstip`, `data.action`, `data.dstport`, etc.)

### 2.2 Ce qui est actuellement visible

La grande majorité des alertes correspondent à :

| Champ | Valeur typique (lab) | Interprétation |
|-------|----------------------|----------------|
| Source | IP du firewall (côté WAN) | OPNsense lui-même |
| Destination | `…255` (broadcast) | Broadcast segment WAN |
| Port | 9431/UDP | Trafic type STUN/TURN / broadcast |
| Action | block | Bloqué par règle WAN |
| Règle Wazuh | 87702 | Multiple blocks from same source |

**Conclusion :**  
On observe surtout du **bruit interne** (broadcast bloqué sur le WAN), pas encore de vraies tentatives externes sur l’IP publique lab (`192.168.5.z`).

Normal dans un lab résidentiel peu exposé.

---

## 3. Filtres Discover recommandés

### 3.1 Filtres de base (à sauvegarder)

| Nom du Saved Search | Conditions principales | Usage |
|---------------------|------------------------|-------|
| OPNsense – All filterlog | `predecoder.hostname: OPNsense.internal` AND `predecoder.program_name: filterlog` | Vue globale |
| OPNsense – Blocks only | + `data.action: block` | Uniquement les blocages |
| OPNsense – Public IP activity | + `data.dstip: 192.168.5.z` | Trafic vers l’IP publique lab |
| OPNsense – Exclude broadcast noise | Exclure destination broadcast et port 9431 | Nettoyer le bruit |

**Important :** toujours utiliser le bouton **Save** (Saved Search), pas seulement le pin — sinon les filtres disparaissent.

### 3.2 Comment créer un filtre multi-conditions

Dans Discover :

1. **+ Add filter**  
2. Une condition à la fois  
3. Les conditions s’additionnent en **ET**  
4. Jeu correct → **Save** avec un nom clair  

---

## 4. Alertes intelligentes (préparation)

| Règle cible | Détection | Niveau visé | Intérêt démo |
|-------------|-----------|-------------|--------------|
| SSH Brute-Force | Plusieurs blocks sur port 22 vers l’IP publique | 10–12 | Très fort |
| Port Scan on Public IP | Volume élevé de blocks depuis une même source | 10 | Excellent |
| Suspicious traffic to WAN | Ports sensibles (22, 3389, 445, 21…) | 8–10 | Très bon |
| Multiple blocks from same source | Déjà présent (règle **87702**) | 10 | Déjà actif |

Ces règles se placent typiquement dans `/var/ossec/etc/rules/local_rules.xml`.

**État lab :** la 87702 fonctionne déjà. Les autres demandent plus de volume réel **ou** des simulations contrôlées.

---

## 5. Simulations d’attaque (étape future)

Pour une fiche démontrable :

| Test | Outil | Source recommandée |
|------|-------|--------------------|
| Port scan | `nmap` | Machine externe lab ou VPS **autorisé** |
| Brute-force SSH | outil / script dédié | Source contrôlée |
| Connexions suspectes | `nc` / `hping3` | Périmètre lab uniquement |

**Règles d’or :**

- rester strictement dans le périmètre du lab ;  
- documenter chaque test ;  
- **ne jamais** scanner Internet de façon non autorisée.

---

## 6. Recommandations immédiates

1. Créer et **sauvegarder** les Saved Searches (§ 3)  
2. Exclure le bruit broadcast / port 9431 pour une vue plus propre  
3. Laisser la règle 87702 active (corrélation déjà visible)  
4. Planifier les simulations dans une session dédiée  
5. Captures des Saved Searches pour le portfolio  

---

## 7. Suite

- Fiche **32** : agent Wazuh sur AlphaDeck (recommandé)  
- Ou custom rules une fois les simulations réalisées  

---

**Statut fiche 31A :**  
Socle de filtres et compréhension des logs en place.  
Alertes intelligentes préparées — en attente de volume / simulations pour validation complète.

**Document portfolio – 15 août 2026**
