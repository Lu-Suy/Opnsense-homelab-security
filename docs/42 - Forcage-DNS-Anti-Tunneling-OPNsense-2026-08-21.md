# 42 – Forçage DNS / Anti-Tunneling DNS (OPNsense)

**Date :** 21 août 2026  
**Statut :** Validé sur LAN + OPT1 (tests réussis) – OPT2 règles en place  
**Auteur :** Travail conjoint (lab Bastion Godmode / Horus AIS)  
**Référence inspiration :** Méthodologie type Éric Dupaud (documentation détaillée des étapes, difficultés, validations)

---

## 1. Objectif

Réduire fortement la surface d’attaque **DNS Tunneling** (MITRE T1071.004 + T1048.003) sur l’infrastructure.

Un malware ou un acteur post-compromission ne doit plus pouvoir :
- Exfiltrer des données ou maintenir un C2 en interrogeant librement des résolveurs publics (8.8.8.8, 9.9.9.9, 1.1.1.1…)
- Utiliser des sous-domaines à haute entropie, un volume anormal de requêtes, ou des types de records particuliers (TXT, NULL, etc.)

**Principe retenu :**  
Toutes les machines des zones internes doivent être **forcées** à utiliser Unbound (résolveur local d’OPNsense).  
Aucun trafic DNS classique (port 53 TCP/UDP) ne doit pouvoir sortir directement vers Internet.

---

## 2. État initial (avant intervention)

D’après l’export de configuration et les captures :

- Unbound en mode **récursif pur** (pas de Query Forwarding, pas de DNS over TLS upstream configuré).
- Règles **Pass** explicites autorisant le DNS sortant vers l’alias `DNS_External` (ports 53, 853 et 443) sur LAN, OPT1 et OPT2.
- **Aucune** règle Destination NAT de forçage DNS.
- Accès des clients vers Unbound (This Firewall:53) **non explicitement autorisé** → le trafic était bloqué par le « Block All » final (découvert en Live View).
- AlphaDeck avait des DNS codés en dur : `9.9.9.9` + `1.1.1.1`.

Conséquence : un client pouvait librement interroger n’importe quel résolveur public → canal de tunneling ouvert.

---

## 3. Tentatives avec Destination NAT (Port Forward / Redirect)

### 3.1 Première approche

Création d’une règle **Destination NAT** sur l’interface LAN :

| Champ                    | Valeur essayée                          |
|--------------------------|-----------------------------------------|
| Interface                | igc2_LAN                                |
| Protocol                 | TCP/UDP                                 |
| Source                   | igc2_LAN network                        |
| Destination              | any (ou !This Firewall)                 |
| Destination port         | 53                                      |
| Redirect target IP       | 10.0.0.1 puis 127.0.0.1                 |
| Redirect target port     | 53                                      |
| Firewall rule            | Pass                                    |
| Description              | Force DNS to Unbound - LAN              |

### 3.2 Difficultés rencontrées

1. **Accès à Unbound bloqué**  
   Live View montrait clairement des `block` sur `10.0.0.10 → 10.0.0.1:53`.  
   Il a fallu d’abord créer une règle **Pass** explicite « Allow DNS to Unbound from LAN » (This Firewall:53).

2. **Interaction avec les règles Pass existantes**  
   Les règles « Allow DNS Outbound » vers `DNS_External:53` continuaient de laisser passer le trafic avant (ou en parallèle) du redirect.

3. **Comportement instable du rdr**  
   Même après désactivation des Pass sortants et changement de target IP (10.0.0.1 → 127.0.0.1), les `nslookup` vers 8.8.8.8 / 9.9.9.9 continuaient de recevoir des réponses directes des résolveurs externes.

4. **Complexité de debug**  
   Le mélange Destination NAT + règles de filtre associées + règles Pass historiques rendait le comportement difficile à prédire.

**Décision :** abandon de la méthode Destination NAT pour cette phase.  
Méthode plus simple et déterministe retenue : **Allow local + Block external**.

---

## 4. Méthode finale retenue

Sur chaque interface (LAN, OPT1, OPT2) :

1. **Pass** – Autoriser explicitement le DNS vers Unbound  
   - Destination = **This Firewall**  
   - Port = 53 (domain)

2. **Block** – Interdire tout autre DNS  
   - Destination = **any**  
   - Port = 53 (domain)

**Ordre critique** (évaluation de haut en bas) :

```
Allow DNS to Unbound from <zone>     ← Pass (This Firewall:53)
Block external DNS (force Unbound)   ← Block (any:53)
```

Les anciennes règles « Allow DNS Outbound » (port 53 vers DNS_External) ont été **désactivées** sur chaque interface.

---

## 5. Règles exactes appliquées

### 5.1 Interface LAN (igc2_LAN – 10.0.0.0/24)

| Ordre | Action | Source              | Destination     | Port | Description                              | État     |
|-------|--------|---------------------|-----------------|------|------------------------------------------|----------|
| 1     | Pass   | igc2_LAN network    | This Firewall   | 53   | Allow DNS to Unbound from LAN            | Actif    |
| 2     | Block  | igc2_LAN network    | any             | 53   | Block external DNS (force Unbound)       | Actif    |

- Ancienne règle « Allow DNS Outbound » (DNS_External:53) → **désactivée**
- Direction : in
- Log : activé sur les deux règles

### 5.2 Interface OPT1 (OPT1_Bastion – 10.0.10.0/24)

| Ordre | Action | Source                | Destination     | Port | Description                                      | État     |
|-------|--------|-----------------------|-----------------|------|--------------------------------------------------|----------|
| 1     | Pass   | OPT1_Bastion network  | This Firewall   | 53   | Allow DNS to Unbound from OPT1                   | Actif    |
| 2     | Block  | OPT1_Bastion network  | any             | 53   | Block external DNS (force Unbound) - OPT1        | Actif    |

- Ancienne règle « DNS Outbound (Do53) » → **désactivée**

### 5.3 Interface OPT2 (OPT2_GX10 – 10.0.20.0/24)

| Ordre | Action | Source             | Destination     | Port | Description                                      | État     |
|-------|--------|--------------------|-----------------|------|--------------------------------------------------|----------|
| 1     | Pass   | OPT2_GX10 network  | This Firewall   | 53   | Allow DNS to Unbound from OPT2                   | Actif    |
| 2     | Block  | OPT2_GX10 network  | any             | 53   | Block external DNS (force Unbound) - OPT2        | Actif    |

- Ancienne règle « DNS Outbound (Do53 classique) » → **désactivée**

---

## 6. Tests de validation

### 6.1 AlphaDeck (LAN – 10.0.0.10)

Après changement des DNS manuels de la machine vers `10.0.0.1` :

```text
nslookup google.com
→ Server: OPNsense.internal / Address: 10.0.0.1   (OK)

nslookup google.com 8.8.8.8
→ timeout                                          (Bloqué)

nslookup google.com 9.9.9.9
→ timeout                                          (Bloqué)
```
![[Pasted image 20260821152822.png]]
### 6.2 G4 / Bastion (OPT1 – 10.0.10.11)

Installation de `bind-utils` (après forçage temporaire de `/etc/resolv.conf` vers 10.0.10.1) :

```text
dig @10.0.10.1 google.com +short
→ 142.251.39.206                                   (OK)

dig @8.8.8.8 google.com +short
→ no servers could be reached                      (Bloqué)

dig @9.9.9.9 google.com +short
→ communications error … timed out                 (Bloqué)
```
![[Pasted image 20260821153233.png]]
### 6.3 OPT2

Règles créées et appliquées.  
Test client non réalisé dans la session (pas de machine de test dédiée au moment de la validation).

---

## 7. Points d’attention & effets secondaires

| Point | Détail | Action / Suivi |
|-------|--------|----------------|
| DNS hard-codés | AlphaDeck (et potentiellement d’autres machines) avaient 9.9.9.9 / 1.1.1.1 en dur | À corriger via DHCP ou configuration manuelle → 10.0.0.1 / 10.0.10.1 |
| DoT (port 853) | Règles Pass encore actives vers DNS_External:853 | Canal encore ouvert (à traiter) |
| DoH (port 443) | Règles Pass encore actives (et trafic HTTPS général) | Difficile à bloquer sans casser la navigation ; à surveiller |
| Point de défaillance unique | Si Unbound tombe, plus de résolution DNS | Surveiller le service Unbound + logs |
| Logs | Les règles Block sont loggées → volume de logs potentiellement plus élevé | Surveiller la rotation / taille des logs firewall |
| Norton VPN (AlphaDeck) | Intercepte le DNS quand actif | À désactiver pendant les tests / à gérer séparément |
| Floating rules | Possible alternative plus élégante pour multi-interfaces | Non retenue pour l’instant (préférence gestion par zone) |

---
## 8. Suite de la soirée – Unbound upstream DoT + Rate-limit

Après validation du forçage client (sections 1 à 7), nous avons enchaîné sur le durcissement d’Unbound lui-même.

### 8.1 Objectif de cette suite

- Chiffrer les requêtes DNS sortantes d’Unbound (DNS over TLS)
    
- Limiter le volume de requêtes par IP client (protection supplémentaire anti-tunneling)
    

### 8.2 Configuration DNS over TLS (upstream)

**Menu :** Services → Unbound DNS → DNS over TLS

Quatre entrées créées (Domain laissé **vide** = catch-all) :

|   |   |   |   |   |
|---|---|---|---|---|
|Enabled|Domain|Server IP|Server Port|Description|
|Oui|_(vide)_|1.1.1.1|853|Cloudflare primaire|
|Oui|_(vide)_|1.0.0.1|853|Cloudflare secondaire|
|Oui|_(vide)_|9.9.9.9|853|Quad9 primaire|
|Oui|_(vide)_|149.112.112.112|853|Quad9 secondaire|

**Note importante :**  
Dans l’interface OPNsense, le champ « Domain » doit rester vide pour un forwarder global.  
La vérification du certificat TLS (Verify CN) est gérée automatiquement ou via les noms fournis dans la description / configuration interne.

« Use System Nameservers » reste **décoché**.

Après Apply, la résolution continue de fonctionner normalement via Unbound.

### 8.3 Rate-limit Unbound (`ip-ratelimit`)

L’onglet Advanced de la version en place **ne propose pas** de champ GUI pour `ip-ratelimit`.

**Méthode retenue :** fichier de configuration personnalisé inclus automatiquement par Unbound.

Commandes exactes exécutées sur OPNsense :

```bash
mkdir -p /var/unbound/etc
echo 'server:' > /var/unbound/etc/ratelimit.conf
echo '  ip-ratelimit: 150' >> /var/unbound/etc/ratelimit.conf
configctl unbound restart
```

Contenu final du fichier `/var/unbound/etc/ratelimit.conf` :

```
server:
  ip-ratelimit: 150
```

**Effet :**  
Chaque adresse IP cliente est limitée à environ 150 requêtes DNS par seconde.  
Au-delà, les requêtes excessives sont droppées (protection contre les volumes anormaux typiques du tunneling).

### 8.4 Validation après DoT + rate-limit

|   |   |
|---|---|
|Test|Résultat|
|`nslookup google.com` (via Unbound)|OK – résolution correcte|
|`dig @10.0.10.1 google.com +short`|OK|
|Tentative directe `dig @8.8.8.8` ou vers IP externe:53|Timeout / no servers could be reached (conforme au forçage)|

### 8.5 Difficultés rencontrées sur cette partie

- Absence de champ « Custom options » dans l’onglet Advanced → passage par fichier `/var/unbound/etc/`.
    
- Heredoc (`<< EOF`) capricieux sur la console OPNsense → utilisation de `echo` ligne par ligne plus fiable.
    
- Attention à l’IP Quad9 secondaire : bien utiliser `149.112.112.112` (et non 142.x).
## 9. Synthèse globale de la soirée

|   |   |   |   |   |
|---|---|---|---|---|
|Zone|Allow This Firewall:53|Block any:53|Ancienne règle 53 désactivée|Tests validés|
|LAN|Oui|Oui|Oui|Oui (AlphaDeck)|
|OPT1|Oui|Oui|Oui|Oui (G4)|
|OPT2|Oui|Oui|Oui|Règles OK|

|   |   |   |
|---|---|---|
|Composant Unbound|Statut|Détail|
|Upstream DNS over TLS|✅ Actif|Cloudflare + Quad9 (port 853)|
|Rate-limit (`ip-ratelimit`)|✅ Actif|150 requêtes/s par IP cliente|

Le canal DNS Tunneling **classique (port 53)** est fermé sur les trois zones, et Unbound chiffre désormais ses requêtes sortantes tout en limitant les volumes.

---

## 10. Prochaines étapes restantes

1. **Push DNS via DHCP** : s’assurer que les clients reçoivent automatiquement 10.0.0.1 / 10.0.10.1 / 10.0.20.1.
    
2. **Décision sur le port 853** : bloquer ou non le DoT direct depuis les clients.
    
3. **Surveillance** : règles Suricata / Wazuh sur les volumes et patterns DNS anormaux.
    
4. **Test OPT2** : valider avec une machine réelle du réseau 10.0.20.0/24.
    
5. Ingestion logs BunkerWeb/Docker → Wazuh et alertes email (priorités suivantes du lab).
    

---

**Fin de la fiche 42 (mise à jour 21 août 2026 soir)**  
Document consolidé : forçage client + Unbound DoT + rate-limit.