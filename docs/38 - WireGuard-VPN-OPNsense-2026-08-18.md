# 38 – Installation et configuration WireGuard VPN sur OPNsense  
**Accès distant sécurisé (Road Warrior) – Split-tunnel**

**Projet :** Bastion Godmode / Horus AIS  
**Machine :** OPNsense (HP EliteDesk 800 G2 SFF – i5-6500 / 16 Go RAM)  
**Date :** 18 août 2026  
**Dernière mise à jour :** 18 août 2026 (soir – validation live)  
**Auteur :** Ludovic + équipe  
**Lien avec docs précédentes :** 02-network, 04 / 04b (Firewall Rules), 11b (Port-Forwarding FAI), 14 (Inventaire Hardware), 30-32 (Wazuh)

---

## 1. Objectif

Mettre en place un accès distant sécurisé de type **Road Warrior** vers le réseau interne (`10.0.0.0/8`) depuis :

- AlphaDeck (Windows 10 – machine d’administration principale)
- Samsung Galaxy Note20 Ultra (smartphone principal)

**Principes retenus :**
- Split-tunnel uniquement (seul le trafic vers `10.0.0.0/8` passe dans le tunnel)
- Authentification forte (clés asymétriques + Pre-Shared Key)
- Surface d’attaque minimale
- Documentation complète et reproductible (méthodologie projet structurée et versionnée)

WireGuard a été choisi pour sa simplicité, ses performances (kernel-level), sa faible surface d’attaque et son comportement « silencieux » face aux probes non authentifiés.

**Pourquoi WireGuard plutôt qu’OpenVPN :**  
WireGuard présente une surface de code extrêmement réduite (~4 000 lignes vs >100 000 pour OpenVPN), des performances kernel-level, une configuration simple et un comportement silencieux face aux probes non authentifiés. Cela réduit significativement la surface d’attaque et le bruit de fingerprinting. Pour un usage Road Warrior avec un nombre limité de peers de confiance, il offre un excellent ratio sécurité / simplicité / performance. OpenVPN reste pertinent pour des besoins de certificats X.509 complexes ou de fallback TCP, ce qui n’est pas notre cas.

---

## 2. Décisions techniques figées

| Élément                    | Valeur finale                  | Justification |
|----------------------------|--------------------------------|---------------|
| Réseau tunnel              | **10.8.8.0/24**                | Plage isolée, mémorable, hors des réseaux existants (10.0.0.0/24, 10.0.10.0/24, 10.0.20.0/24) |
| IP serveur (OPNsense)      | **10.8.8.1/24**                | Adresse de l’instance WireGuard |
| AlphaDeck (peer)           | **10.8.8.2/32**                | IP fixe du client principal |
| Galaxy Note20 (peer)       | **10.8.8.3/32**                | IP fixe du smartphone |
| Port d’écoute              | **51888/UDP**                  | Port non standard (moins scanné que 51820) |
| Type de tunnel             | **Split-tunnel**               | Seul le trafic vers le réseau interne passe dans le VPN |
| AllowedIPs (côté clients)  | **10.0.0.0/8**                 | Accès à tout le réseau privé interne (LAN + OPT1 + OPT2) |
| PersistentKeepalive        | **25**                         | Valeur recommandée officielle WireGuard pour clients derrière NAT / mobile |
| MTU                        | **1420** (défaut)              | Standard OPNsense ; ajustable si fragmentation |
| Interface instance         | `wg0`                          | Nom retenu |
| DNS public                 | `vpn.horus-ais.com`            | Enregistrement A en mode **DNS only** (nuage gris Cloudflare) |
| Public Key serveur         | `L8VuzAAMtAF6erxjsrGK1ky0QzP9XHeHSsk6ZgwuZlE=` | Clé générée sur OPNsense |

**Note importante sur les clés :**  
Les Private Keys des clients et les Pre-Shared Keys (PSK) ne sont **jamais** stockées dans la documentation. Elles restent uniquement dans le gestionnaire de mots de passe et sur les appareils concernés.

---

## 3. Prérequis & capacité hardware

### Hardware
- **OPNsense** : HP EliteDesk 800 G2 SFF – Intel Core i5-6500 (4C/4T) + 16 Go RAM  
  → Largement suffisant. WireGuard est extrêmement léger (overhead négligeable même avec plusieurs dizaines de peers).
- Clients : AlphaDeck (Windows 10) + Galaxy Note20 Ultra (Android)

### Logiciel / Accès
- OPNsense à jour (plugin WireGuard / os-wireguard disponible)
- Accès GUI OPNsense depuis AlphaDeck
- Accès à la zone DNS de `horus-ais.com` (Cloudflare)
- Accès à l’interface de la box FAI (Nordnet) pour le port-forward
- Application officielle WireGuard installée sur AlphaDeck et sur le Note20

### Différence fondamentale avec Cloudflare Tunnel
| Aspect                  | Cloudflare Tunnel                          | WireGuard (ce document)                  |
|-------------------------|--------------------------------------------|------------------------------------------|
| Usage                   | Sites web publics (HTTP/HTTPS)             | Accès distant administrateur / interne   |
| Protocole               | HTTP/HTTPS sortant                         | UDP inbound                              |
| Chemin                  | G4 → Cloudflare → Internet                 | Internet → Box FAI → OPNsense            |
| DNS                     | Proxied (orange) possible                  | **DNS only (gris)** obligatoire          |
| Exposition              | Aucun port ouvert en inbound               | Port UDP 51888 ouvert                    |
| Sécurité                | Cloudflare WAF + Tunnel                    | Clés crypto + PSK + règles firewall      |

Les deux coexistent parfaitement : les sites (`horus-ais.com`, `mrdoolux.horus-ais.com`) restent derrière Cloudflare Tunnel, tandis que `vpn.horus-ais.com` pointe directement vers l’IP publique pour WireGuard.

---

## 4. Architecture réseau du tunnel

```
Internet
   │
   │  UDP 51888
   ▼
Box FAI (Nordnet)  ── port-forward ──► OPNsense WAN (192.168.5.244)
                                              │
                                              │ WireGuard instance wg0
                                              │ 10.8.8.1/24
                                              ▼
                                    Clients (10.8.8.2 / 10.8.8.3)
                                              │
                                              │ AllowedIPs = 10.0.0.0/8
                                              ▼
                                    Réseau interne complet
                                    (LAN 10.0.0.0/24 + OPT1 10.0.10.0/24 + OPT2 10.0.20.0/24)
```

---

## 5. Process de déploiement (chronologique)

### 5.1 Création de l’instance WireGuard (serveur)

1. **VPN → WireGuard → Instances** → **+**
2. Paramètres :

| Champ              | Valeur                  |
|--------------------|-------------------------|
| Enabled            | Coché                   |
| Name               | `wg0` (ou `Bastion-WG`) |
| Listen Port        | `51888`                 |
| Tunnel Address     | `10.8.8.1/24`           |
| MTU                | `1420`                  |
| Peers              | (vide pour l’instant)   |

3. Générer les clés avec l’icône engrenage (Public / Private Key).
4. **Save** → **Apply**.
5. Rouvrir l’instance et **copier la Public Key** du serveur (nécessaire pour les clients).

**Effet :**  
Création de l’interface logique `wg0` et génération de la paire de clés asymétriques du serveur. Aucun trafic n’est encore accepté.

---

### 5.2 Création des peers (clients)

Utiliser de préférence le **Peer Generator** (génère aussi le QR code pour le téléphone).

#### Peer AlphaDeck
| Champ                | Valeur                          |
|----------------------|---------------------------------|
| Enabled              | Coché                           |
| Name                 | `AlphaDeck`                     |
| Public key           | Public Key générée sur AlphaDeck |
| Pre-shared key       | Générée via engrenage (recommandé) |
| Allowed IPs          | `10.8.8.2/32`                   |
| Endpoint address     | *vide* (road-warrior)           |
| Endpoint port        | `51888` (ou vide)               |
| Instances            | `wg0`                           |
| Keepalive interval   | `25`                            |
**Comment obtenir la Public Key d’AlphaDeck ?**

Sur AlphaDeck (Windows) :

1. Installe l’application officielle WireGuard.
2. Clique sur « Add Tunnel » → « Add empty tunnel ».
![[Pasted image 20260818131324.png]]
3. Ça génère automatiquement une Private Key + Public Key.
4. Copie la **Public Key** et colle-la dans le champ « Public key » de OPNsense.
![[Pasted image 20260818131025.png]]

#### Peer Note20
| Champ                | Valeur                          |
|----------------------|---------------------------------|
| Enabled              | Coché                           |
| Name                 | `Note20`                        |
| Public key           | Public Key générée sur le téléphone |
| Pre-shared key       | Nouvelle PSK générée            |
| Allowed IPs          | `10.8.8.3/32`                   |
| Endpoint address     | *vide*                          |
| Endpoint port        | `51888`                         |
| Instances            | `wg0`                           |
| Keepalive interval   | `25`                            |

**Point critique :**  
Côté **serveur**, le champ **Allowed IPs** d’un peer = uniquement l’adresse IP du client dans le tunnel (`/32`).  
Le split-tunnel (`10.0.0.0/8`) se configure **côté client**.

Après création des peers → éditer l’instance `wg0` et s’assurer que les deux peers sont bien sélectionnés dans le champ **Peers**.
![[Pasted image 20260818133813.png]]
---

### 5.3 Activation du service WireGuard

1. **VPN → WireGuard → General**
2. Cocher **Enable WireGuard**
3. **Apply**
![[Pasted image 20260818133757.png]]
**Effet :**  
Le service démarre, l’interface `wg0` devient active. Les handshakes peuvent maintenant être acceptés si les clés correspondent.

---

### 5.4 Règle Firewall WAN (indispensable)

Sans cette règle, les paquets UDP 51888 sont droppés par le firewall.

**Firewall → Rules → WAN → +**

| Champ                  | Valeur                          |
|------------------------|---------------------------------|
| Action                 | Pass                            |
| Interface              | WAN                             |
| Direction              | in                              |
| TCP/IP Version         | IPv4                            |
| Protocol               | UDP                             |
| Source                 | Any                             |
| Destination            | This Firewall **ou** WAN address |
| Destination port       | 51888                           |
| Description            | WireGuard VPN                   |
| Log                    | Coché (recommandé)              |

**Save** → **Apply Changes**.
![[Pasted image 20260818133327.png]]
**Effet :**  
Autorise uniquement le protocole UDP sur le port 51888 en entrée sur le WAN. Tout le reste reste bloqué par le default deny.

---

### 5.5 DNS et Port-Forward (accès depuis l’extérieur)

#### Enregistrement DNS (Cloudflare)
- Type : **A**
- Name : `vpn`
- IPv4 address : **IP publique actuelle de la box FAI** (ex. IP_PUBLIC au moment du déploiement)
- Proxy status : **DNS only** (nuage **gris**)

**Vérification (depuis AlphaDeck – PowerShell) :**
```powershell
Resolve-DnsName vpn.horus-ais.com
# ou
nslookup vpn.horus-ais.com
```
![[Pasted image 20260818143012.png]]
#### Port-Forward sur la box FAI (Nordnet)
- Protocole : **UDP**
- Port externe : **51888**
- IP de destination : **192.168.5.244** (WAN d’OPNsense)
- Port interne : **51888**
![[Pasted image 20260818142307.png]]
**Effet :**  
Les paquets UDP arrivant sur l’IP publique:51888 sont redirigés vers OPNsense. WireGuard prend ensuite le relais.

---

### 5.6 Configurations clients

#### AlphaDeck (Windows – application officielle WireGuard)

```ini
[Interface]
PrivateKey = REMPLACER_PAR_PRIVATE_KEY_ALPHADESK
Address = 10.8.8.2/32
DNS = 10.8.8.1

[Peer]
PublicKey = L8VuzAAMtAF6erxjsrGK1ky0QzP9XHeHSsk6ZgwuZlE=
PresharedKey = REMPLACER_PAR_PSK_ALPHADESK
AllowedIPs = 10.0.0.0/8
Endpoint = vpn.horus-ais.com:51888
PersistentKeepalive = 25
```
![[Pasted image 20260818134731.png]]
#### Galaxy Note20 (application officielle WireGuard)

Même structure, avec :
- `Address = 10.8.8.3/32`
- PrivateKey et PresharedKey du peer Note20

**Effet des paramètres clients :**
- `AllowedIPs = 10.0.0.0/8` → split-tunnel (seul le trafic interne passe dans le VPN)
- `PersistentKeepalive = 25` → maintient le mapping NAT actif
- `DNS = 10.8.8.1` → résolution DNS via OPNsense pendant la connexion VPN
![[WhatsApp Image 2026-08-19 at 11.47.56 AM.jpeg]]
---

### 5.7 Règle d’accès interne (Floating / WireGuard Group) + Assignation d’interface

Au moment du déploiement initial, une règle Floating a été créée :

**Firewall → Rules → Floating → +**

| Champ                  | Valeur                                      |
|------------------------|---------------------------------------------|
| Action                 | Pass                                        |
| Interface              | **WireGuard (Group)**                       |
| Direction              | in                                          |
| TCP/IP Version         | IPv4                                        |
| Protocol               | any                                         |
| Source                 | `10.8.8.0/24`                               |
| Destination            | `10.0.0.0/8`                                |
| Description            | Allow WireGuard clients internal access     |
| Log                    | Coché                                       |
![[Pasted image 20260819113716.png]]
**Save** → **Apply Changes**.

**Effet :**  
Les clients connectés au tunnel peuvent atteindre l’ensemble du réseau privé `10.0.0.0/8` (G4, Wazuh, BunkerWeb, OPNsense GUI, etc.).

**Assignation de l’interface (réalisée) :**  
L’interface `wg0` a ensuite été assignée via **Interfaces → Assignments** (Enable + Description « WireGuard » / IPv4 Configuration Type = None).  
Cela permet d’avoir un alias propre et d’écrire des règles directement sur l’interface à l’avenir.

### 5.8 Règle Normalization / MSS Clamping (réalisée le 18 août 2026)

Une règle de **Normalization** a été ajoutée pour éviter la fragmentation des paquets TCP à l’intérieur du tunnel WireGuard.

**Firewall → Settings → Normalization → +**

|Champ|Valeur|
|---|---|
|Interface|WireGuard (Group)|
|Direction|any|
|Protocol|any|
|Max MSS|**1380**|
|Description|WireGuard MSS Clamping|
![[Pasted image 20260818180137.png]]
**Save → Apply.**

**Effet :** WireGuard réduit le MTU effectif à 1420. Sans clamping, certains flux TCP (sessions longues, dashboards, transferts) peuvent fragmenter ou devenir instables. La valeur **1380** correspond à : MTU WireGuard (1420) − 40 octets d’en-têtes IP + TCP.

Cette règle force les clients à négocier une taille de segment TCP compatible avec le tunnel. C’est une recommandation officielle OPNsense pour les déploiements Road Warrior.

**Statut :** Règle active et appliquée.

---

## 6. Tests de validation (état au 18 août 2026 soir)

### Checklist de validation

| Test                                      | Méthode                                      | Résultat attendu                  | Statut |
|-------------------------------------------|----------------------------------------------|-----------------------------------|--------|
| Résolution DNS `vpn.horus-ais.com`        | `Resolve-DnsName` / `dig`                    | IP publique de la box             | ✅ Validé |
| Handshake visible                         | VPN → WireGuard → Status                     | Handshake récent + transfert data | ✅ Validé (live 4G) |
| Accès depuis 4G (Note20)                  | Wi-Fi off + données mobiles + tunnel actif   | Accès interne OK                  | ✅ Validé |
| Accès Wazuh Dashboard                     | `https://10.0.10.11:8443`                    | Page de login                     | ✅ Validé |
| Accès BunkerWeb / sites                   | via **noms de domaine** (résolution)         | Sites accessibles                 | ✅ Validé |
| SSH vers G4                               | `ssh hyper_doo@10.0.10.11`                   | Connexion OK                      | ⏳ Non testé |
| Interface `wg0` assignée                  | Interfaces → Assignments                     | Interface visible + alias         | ✅ Réalisé |

**Note :**  
La configuration est **opérationnelle**. Handshake et accès aux services critiques (Wazuh + BunkerWeb via résolution de noms) ont été validés en live depuis le Galaxy Note20 en 4G. L’accès SSH depuis le tunnel n’a pas encore été testé explicitement.
![[Pasted image 20260818170411.png]]

![[WhatsApp Image 2026-08-19 at 11.44.30 AM.jpeg]]

![[WhatsApp Image 2026-08-19 at 11.44.54 AM.jpeg]]
---

## 7. Analyse de sécurité – Exposition du port UDP 51888

### Ce qui est exposé
Un scanner Internet peut détecter que le port UDP 51888 est ouvert sur l’IP publique (grâce au port-forward).

### Ce qui se passe réellement

1. L’attaquant envoie des paquets vers le port.
2. Les paquets arrivent sur OPNsense.
3. WireGuard examine le paquet.
4. Si la clé publique du client **n’est pas connue** ou si le handshake cryptographique échoue → **WireGuard ignore purement et simplement le paquet** (aucune réponse, aucun log verbeux par défaut).

C’est l’une des forces majeures de WireGuard : il est extrêmement discret.

### Tableau de risque

| Couche                         | Protection                              | Niveau de risque |
|--------------------------------|-----------------------------------------|------------------|
| Port ouvert sur la box FAI     | Visible par scan                        | Faible           |
| Port-forward vers OPNsense     | Existe                                  | Moyen            |
| Authentification WireGuard     | Clés asymétriques + PSK                 | **Très fort**    |
| Firewall OPNsense              | Règles strictes + logging               | Fort             |
| Absence de réponse aux probes  | WireGuard silencieux                    | Excellent        |

**Conclusion :**  
Le port est techniquement ouvert et visible, mais la « porte » derrière est extrêmement bien fermée. Le seul vrai risque résiduel serait une fuite de clé privée client ou une vulnérabilité 0-day dans WireGuard lui-même (extrêmement rare et rapidement patchée).

Comparé à OpenVPN classique, WireGuard est nettement moins bavard et plus difficile à fingerprint.

---

## 8. Accès possibles via le tunnel (état actuel)

Grâce à `AllowedIPs = 10.0.0.0/8` + la règle Floating :

| Ressource                          | Accessible via WireGuard ? | Comment |
|------------------------------------|----------------------------|---------|
| G4 (10.0.10.10 / 10.0.10.11)       | Oui                        | HTTP, HTTPS, SSH, Wazuh |
| Wazuh Dashboard                    | Oui                        | `https://10.0.10.11:8443` |
| BunkerWeb / sites                  | Oui                        | Via domaines ou IP |
| OPNsense GUI / SSH                 | Oui                        | 10.0.10.1 |
| Autres machines 10.0.x.x           | Oui (selon règles locales) | — |
![[Pasted image 20260818170211.png]]
**Point d’attention :**  
Les règles OPT1 actuelles (accès BunkerWeb 80/443 et Wazuh) sont historiquement limitées à AlphaDeck depuis le LAN. La Floating rule globale permet l’accès depuis le tunnel, mais une durcification plus granulaire (règles explicites Source = 10.8.8.0/24) est recommandée à moyen terme.

---

## 9. Améliorations futures recommandées


2. **Test SSH depuis le tunnel**  
   Valider explicitement `ssh hyper_doo@10.0.10.11` (ou l’IP Manager) depuis un client WireGuard.

3. **Règles plus granulaires**  
   - Autoriser explicitement 10.8.8.0/24 → 10.0.10.10:80,443 (BunkerWeb)  
   - Autoriser 10.8.8.0/24 → 10.0.10.11:8443 (Wazuh)  
   - Restreindre le reste si nécessaire.

4. **Logging & intégration Wazuh**  
   - Activer les logs sur toutes les règles WireGuard.  
   - Faire remonter les logs WireGuard / handshakes vers Wazuh (objectif déjà identifié pour BunkerWeb/Docker).

5. **Rotation périodique des clés**  
   - Bonne pratique annuelle ou après tout incident.

6. **Surveillance de l’IP publique**  
   - Si l’IP publique change → mettre à jour l’enregistrement A `vpn.horus-ais.com` (ou mettre en place un client DynDNS).

---

## 10. Checklist finale au 18 août 2026 (soir)

- [x] Décisions techniques figées (réseau, IPs, port, split-tunnel)
- [x] Instance `wg0` créée + clés générées
- [x] Peers AlphaDeck + Note20 créés avec PSK
- [x] Service WireGuard activé
- [x] Règle Firewall WAN (UDP 51888) créée + loguée
- [x] Enregistrement DNS `vpn.horus-ais.com` (DNS only)
- [x] Port-forward box FAI configuré
- [x] Configurations clients préparées
- [x] Règle Floating d’accès interne créée
- [x] Interface `wg0` assignée
- [x] Handshake validé en live depuis 4G (Galaxy Note20)
- [x] Accès Wazuh Dashboard validé depuis le tunnel
- [x] Accès BunkerWeb validé depuis le tunnel (via résolution de noms de domaine)
- [ ] Accès SSH depuis le tunnel (non testé explicitement)
- [x] Règle Normalization / MSS Clamping (Max MSS 1380) ajoutée
- [ ] Documentation images intégrées dans le vault Obsidian

---

### Documentation officielle de référence
- [WireGuard Road Warrior Setup – OPNsense](https://docs.opnsense.org/manual/how-tos/wireguard-client.html)
- [WireGuard Site-to-Site – OPNsense](https://docs.opnsense.org/manual/how-tos/wireguard-s2s.html)

---

**Document mis à jour le 18 août 2026 (soir) – statut opérationnel validé en live.**  
Prêt pour intégration dans le vault Obsidian et le dépôt GitHub (`docs/38 - WireGuard-VPN-OPNsense.md`).

**Prochaines actions recommandées :**  
1. Tester explicitement l’accès SSH depuis le tunnel.  

