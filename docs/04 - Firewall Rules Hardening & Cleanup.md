
### Analyse des logs WAN du 21/05/2026 (21 mai 08:xx)
- Tout le trafic **rouge** (IGMP depuis 192.168.5.1 vers 224.0.0.1) → bruit légitime du FAI (multicast TV/services).  
- Tout le trafic **vert** (Out + "let out anything from firewall host itself") → OPNsense qui fait ses mises à jour DNS/NTP/HTTPS. **Rien de suspect.**  
- **Conclusion** : Aucun signe d’attaque. Ton firewall est bien configuré et fait son boulot.

### Aliases nécessaires (vérifie qu’ils existent dans Firewall → Aliases)

| Alias               | Type       | Contenu (une par ligne)                                                                         | Description                             |
| ------------------- | ---------- | ----------------------------------------------------------------------------------------------- | --------------------------------------- |
| Alpha_4_Deck        | Host       | 10.0.0.10/24                                                                                    | Admin_Opnsense                          |
| Prodesk_Manager     | Hosts      | 10.0.10.11                                                                                      | Règles SSH Prodesk                      |
| Prodesk_web_service | Host       | 10.0.10.10                                                                                      | Règle WEB Services                      |
| **GX10**            | Hosts      | 10.0.20.10                                                                                      | Règles OPT2 (machine haute performance) |
| **DNS_External**    | Host(s)    | 1.1.1.1 1.0.0.1 9.9.9.9                                                                         | DNS externes fiables                    |
| **NTP_Servers**     | Host(s)    | 0.pool.ntp.org 1.pool.ntp.org 2.pool.ntp.org 3.pool.ntp.org time.cloudflare.com time.google.com | Serveurs NTP                            |
| **STUN_TURN**       | Host(s)    | stun.x.ai turn.x.ai stun.l.google.com stun1.l.google.com stun2.l.google.com                     | STUN/TURN pour Grok Voice & WebRTC      |
| **RFC1918**         | Network(s) | 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16                                                         | Réseaux privés (utile pour bloquer)     |

**Philosophie appliquée** :  
Default Deny partout + règles explicites + Log activé sur tout + règles spécifiques en haut, règles générales en bas.

### 1. Règles Interface **WAN** (21/05/2026)
Aucune règle manuelle nécessaire (comportement par défaut = tout bloquer).
### Règles Interface WAN – Accès Admin d’Urgence (5 juillet 2026)

**Objectif** : Accès administration depuis le réseau local FAI (192.168.5.0/24) pour éviter tout lockout.

| Ordre | Action | Protocole | Source           | Destination   | Port | Description                                  | Log | Statut |
| ----- | ------ | --------- | ---------------- | ------------- | ---- | -------------------------------------------- | --- | ------ |
| 1     | Pass   | TCP       | 192.168.*.*/32 | This Firewall | 443  | Accès HTTPS GUI depuis IP FAI (anti-lockout) | Yes | Active |
| 2     | Pass   | TCP       | 192.168.*.*/32 | This Firewall | 22   | Accès SSH depuis IP FAI (anti-lockout)       | Yes | Active |

**Note** : Règles placées en haut de l’onglet WAN. IP source mise en dur pour maximiser la restriction.
**Note** : Les règles automatiques « let out anything from firewall host itself » restent actives (trafic sortant de l’OPNsense lui-même).

### Backup + Porte d’urgence (à faire tout de suite)

**Actions concrètes** :

- **Backup complet OPNsense** :
    - Dans **System → Configuration → Backups**
    - Faire un backup manuel + activer les backups automatiques (quotidiens).
- **Accès admin sécurisé (anti-lockout)** :
    - Créer une règle Firewall spécifique sur **WAN** (très restrictive) :
        - Source : mon IP publique actuelle (ou une plage /32)
        - Destination : This Firewall
        - Port : 443 (HTTPS) ou 22 (SSH)
        - Description : "Admin Access Emergency"
    - Active aussi l’accès via **Console** physique (HP EliteDesk) en cas de problème.
    - Option forte : WireGuard VPN pour l’admin distant (on le fera après).


![Capture d’écran](../images/Pasted%20image%2020260705213830.png)


### 2. Règles Interface **LAN** (10.0.0.0/24) – Clients de confiance
(Règles à placer sur l’onglet **LAN** dans OPNsense)

Capture d’écran du **Firewall → Rules → LAN**.
Tout est bien appliqué :

- Les règles BunkerWeb (80 + 443) sont maintenant **correctement pointées sur Prodesk_Web_Service** (et non plus sur Manager).
- SSH est **ultra-restreint** à AlphaDeck uniquement.
- J'ai ajouté la règle **Block All** tout en bas (c’est la plus importante pour le durcissement).
- J'ai activé les **logs sur toutes les règles** → parfait pour la traçabilité et Wazuh plus tard.
- L’alias **AlphaDeck** = 10.0.0.10 (même chose que Admin-Workstations) → nickel.

| Ordre | Action | Protocole | Source             | Destination         | Port         | Description                                       | Log | Statut |
| ----- | ------ | --------- | ------------------ | ------------------- | ------------ | ------------------------------------------------- | --- | ------ |
| 1     | Pass   | TCP       | Admin-Workstations | Prodesk_Web_Service | 80, 443      | Accès BunkerWeb (HTTP + HTTPS) depuis LAN         | Yes | Active |
| 2     | Pass   | TCP       | Admin-Workstations | Prodesk_Manager     | 22           | SSH uniquement depuis Alphadeck vers Manager      | Yes | Active |
| 3     | Pass   | TCP/UDP   | LAN net            | DNS_External        | 53, 853, 443 | DNS Outbound (Do53 + DoT + DoH)                   | Yes | Active |
| 4     | Pass   | UDP       | LAN net            | NTP_Servers         | 123          | NTP Outbound                                      | Yes | Active |
| 5     | Pass   | TCP       | LAN net            | *                   | 80, 443      | HTTP/HTTPS Outbound (navigation + mises à jour)   | Yes | Active |
| 6     | Pass   | ICMP      | LAN net            | *                   | *            | ICMP ping (optionnel mais utile)                  | Yes | Active |
| 7     | Pass   | UDP       | LAN net            | STUN_TURN           | 3478, 5349   | STUN/TURN pour Grok Voice                         | Yes | Active |
| 8     | Pass   | UDP       | LAN net            | *                   | 50000-65535  | WebRTC Media Ports (Grok Voice & RTC)             | Yes | Active |
| 9     | Pass   | TCP       | Admin-Workstations | This Firewall       | 22           | SSH AlphaDeck → OPNsense                          | Yes | Active |
| 10    | Block  | *         | LAN net            | *                   | *            | **Block All LAN** (dernière règle – durcissement) | Yes | Active |



![Capture d’écran](../images/Pasted%20image%2020260705174648.png)

« Mise à jour LAN du 21/05/2026 – Block All ajouté + logs activés partout. AlphaDeck alias confirmé (10.0.0.10). »

### 3. Règles Interface OPT1 (10.0.10.0/24) – Bastion Godmode Mise à jour du 21/05/2026 – Configuration actuelle après durcissement.
### Analyse rapide de tes règles OPT1 actuelles (21/05/2026)

**Points excellents :**

- DNS bien séparé (Do53 + DoT + DoH) → très propre.
- NTP dédié.
- HTTP/HTTPS outbound bien présents (séparés, c’est OK).
- **SSH ultra-restreint** : seul AlphaDeck peut joindre le Manager + 2 règles de blocage explicites → parfait.
- ICMP autorisé.
- **Block All** en dernière règle (activé et loggé) → exactement ce qu’on veut.

| Ordre | Action | Protocole | Source    | Destination         | Port | Description                                                 | Log | Statut |
| ----- | ------ | --------- | --------- | ------------------- | ---- | ----------------------------------------------------------- | --- | ------ |
| 1     | Pass   | TCP/UDP   | OPT1 net  | DNS_External        | 53   | DNS Outbound (Do53)                                         | Yes | Active |
| 2     | Pass   | TCP       | OPT1 net  | DNS_External        | 853  | DNS Over TLS (DoT) Outbound                                 | Yes | Active |
| 3     | Pass   | TCP       | OPT1 net  | DNS_External        | 443  | DNS Over HTTPS (DoH) Outbound                               | Yes | Active |
| 4     | Pass   | UDP       | OPT1 net  | NTP_Servers         | 123  | NTP Outbound                                                | Yes | Active |
| 5     | Pass   | TCP       | OPT1 net  | *                   | 443  | HTTPS Outbound (updates, web, Docker, BunkerWeb, etc.)      | Yes | Active |
| 6     | Pass   | TCP       | OPT1 net  | *                   | 80   | HTTP Outbound (updates, web, Docker, etc.)                  | Yes | Active |
| 7     | Pass   | TCP       | AlphaDeck | Prodesk_Manager     | 22   | SSH uniquement depuis Alphadeck vers Manager (OPT1)         | Yes | Active |
| 8     | Block  | TCP       | *         | Prodesk_Web_Service | 22   | Block SSH vers IP Web Service (10.0.10.10)                  | Yes | Active |
| 9     | Block  | TCP       | *         | Prodesk_Manager     | 22   | Block SSH vers Manager depuis n’importe où ailleurs         | Yes | Active |
| 10    | Pass   | ICMP      | OPT1 net  | *                   | *    | ICMP ping (optionnel mais utile)                            | Yes | Active |
| 11    | Block  | *         | OPT1 net  | 10.0.0.0/8          | *    | **Block trafic vers réseaux privés** (évite pivot vers LAN) | Yes | Active |
| 12    | Block  | *         | OPT1 net  | 172.16.0.0/12       | *    | **Block trafic vers réseaux privés** (évite pivot)          | Yes | Active |
| 13    | Block  | *         | OPT1 net  | *                   | *    | **Block All OPT1** (dernière règle – durcissement final)    | Yes | Active |



![Capture d’écran](../images/Pasted%20image%2020260705181711.png)

Règles Interface OPT1 (10.0.10.0/24) – Bastion Godmode Mise à jour du 21/05/2026 – Configuration actuelle après durcissement.

### Ce que je te recommande

**Pour OPT1 (Bastion)** — on peut encore améliorer :

Ajoute ces deux règles **juste avant** le Block All :

| Ordre | Action | Protocole | Source   | Destination   | Port | Description                                     |
| ----- | ------ | --------- | -------- | ------------- | ---- | ----------------------------------------------- |
| 10    | Block  | *         | *        | 10.0.0.0/8    | *    | Block trafic vers réseaux privés depuis Bastion |
| 11    | Block  | *         | *        | 172.16.0.0/12 | *    | Block trafic vers réseaux privés depuis Bastion |
| 13    | Block  | *         | OPT1 net | *             | *    | **Block All OPT1**                              |
Ça empêche BunkerWeb (si compromis) d’attaquer le LAN ou OPT2.

**Pour OPT2 (GX10)** : Tes règles actuelles sont suffisantes. Tu peux juste ajouter la même règle de blocage vers réseaux privés si tu veux être cohérent.


### 4. Règles Interface **OPT2** (10.0.20.0/24) – GX10 / Lab isolé (nouvelle)
(Règles à créer sur l’onglet **OPT2** – très restrictives)

**Effet précis de ces règles :**

- La GX10 peut sortir sur Internet pour ses mises à jour et son fonctionnement normal (DNS chiffré + NTP + HTTP/HTTPS).
- **Aucune autre machine** (sauf AlphaDeck) ne peut atteindre la GX10.
- Si la GX10 est un jour compromise, elle **ne pourra pas** atteindre le LAN, OPT1 ou le reste du réseau grâce au Block All en dernier.
- Tout est logué → parfait pour la suite Wazuh.

| Ordre | Action | Protocole | Source       | Destination   | Port | Description                                                 | Log | Statut |
| ----- | ------ | --------- | ------------ | ------------- | ---- | ----------------------------------------------------------- | --- | ------ |
| 1     | Pass   | TCP/UDP   | GX10         | DNS_External  | 53   | DNS Outbound (Do53 classique)                               | Yes | Active |
| 2     | Pass   | TCP       | GX10         | DNS_External  | 853  | DNS Over TLS (DoT) Outbound                                 | Yes | Active |
| 3     | Pass   | TCP       | GX10         | *             | 443  | HTTPS Outbound (updates, Docker, web, DoH, etc.)            | Yes | Active |
| 4     | Pass   | TCP       | GX10         | *             | 80   | HTTP Outbound (updates, Docker, etc.)                       | Yes | Active |
| 5     | Pass   | UDP       | GX10         | NTP_Servers   | 123  | NTP Outbound                                                | Yes | Active |
| 6     | Pass   | ICMP      | GX10         | *             | *    | ICMP ping (optionnel mais utile)                            | Yes | Active |
| 7     | Pass   | TCP       | Alpha_4_Deck | GX10          | 22   | SSH depuis Alphadeck vers GX10 (admin lab – optionnel)      | Yes | Active |
| 8     | Block  | *         | GX10         | 10.0.0.0/8    | *    | **Block trafic vers réseaux privés** (évite pivot LAN/OPT1) | Yes | Active |
| 9     | Block  | *         | GX10         | 172.16.0.0/12 | *    | **Block trafic vers réseaux privés** (évite pivot)          | Yes | Active |
| 10    | Block  | *         | GX10         | *             | *    | **Block All OPT2** (dernière règle – isolation forte)       | Yes | Active |

---
![Capture d’écran](../images/Pasted%20image%2020260705183237.png)

---
Effet global : La GX10 est maintenant fortement isolée tout en restant pleinement fonctionnelle. Si elle est compromise un jour, elle ne pourra pas aller vers le LAN ou OPT1 grâce au Block All.

C’est **clair**, **reproductible** et **durci** .

**Prochaine étape ?**  

### Ce qu’on pourrait encore renforcer (si tu veux aller plus loin)

Voici les points qu’on peut améliorer pour rendre l’isolation **encore plus stricte** :

|Amélioration|Pourquoi c’est utile|Recommandation|
|---|---|---|
|**Bloquer explicitement le trafic entrant** depuis LAN et OPT1 vers GX10 (sauf SSH AlphaDeck)|Empêche tout pivot depuis une autre zone|À ajouter|
|**Limiter les ports sortants** (ex: ne pas autoriser tout le 80/443, mais seulement certains domaines de mise à jour)|Réduit la surface d’attaque si la GX10 est compromise|Optionnel (un peu plus contraignant)|
|**Bloquer le trafic vers les réseaux privés** (10.0.0.0/8, 172.16.0.0/12, etc.) depuis GX10|Empêche la GX10 de parler aux autres machines du réseau local|Très recommandé|
|**Rate limiting** sur les connexions sortantes|Limite les attaques DDoS ou scans sortants si compromise|Optionnel|
|**Anti-spoofing** sur l’interface OPT2|Évite que la GX10 usurpe des IPs|Déjà souvent géré globalement par OPNsense|

## 4. Cleanup-plan – Nettoyage de la Prodesk (21/05/2026)

**Objectif**:  
Faire un vrai ménage avant BunkerWeb multisite + Wazuh. Libérer l’espace, nettoyer les logs, préparer les dossiers pour le SIEM.

**État matériel** : Prodesk 600 Mini G2 – Ryzen 7, 16 Go RAM, SSD 1 To → largement suffisant.

### État avant nettoyage (21/05/2026 – 10:xx)

```bash
# Sortie df -h
Filesystem     Size  Used Avail Use% Mounted on
/dev/sda3      923G  3.2G  873G   1% /

# Sortie du -sh /var/log/*
16K     /var/log/alternatives.log
168K    /var/log/apt
33M     /var/log/journal
16M     /var/log/installer
...
4.0K    /var/log/wazuh   ← déjà créé
````

**Nombre de paquets installés** : 336 (très propre pour une Debian 12).

### Commandes de nettoyage (exécutées en hyper_doo)

**1. Mise à jour + nettoyage paquets**

Bash

```
sudo apt update && sudo apt upgrade -y
sudo apt autoclean
sudo apt autoremove --purge -y
sudo apt clean
```

**Effet** :

- Met tout à jour
- Supprime les paquets obsolètes et les fichiers .deb inutiles dans /var/cache/apt
- Libère de l’espace cache.

**2. Nettoyage des logs système**

Bash

```
sudo journalctl --vacuum-time=2weeks
sudo journalctl --vacuum-size=100M
```

**Effet** :

- Garde seulement les logs des 2 dernières semaines
- Limite /var/log/journal à 100 Mo maximum
- Prépare parfaitement pour Wazuh (on ne garde pas des années de logs inutiles).

**3. Création / sécurisation dossier Wazuh** (déjà fait mais on vérifie)

Bash

```
sudo mkdir -p /var/log/wazuh
sudo chown -R root:adm /var/log/wazuh
sudo chmod -R 750 /var/log/wazuh
```

**Effet** : Dossier prêt pour les logs Wazuh + permissions strictes (root + groupe adm).

### Résultats après nettoyage (à coller ici après exécution)

Bash

```
# À remplir après les commandes
df -h
du -sh /var/log/*
apt list --installed | wc -l
```

**Prochaines étapes après ce cleanup** :

- BunkerWeb multisite + Let’s Encrypt (fichier 11)
- WireGuard (fichier 12)

**Dernière mise à jour** : 21 mai 2026 – Cleanup terminé, Prodesk prête pour les gros services.
