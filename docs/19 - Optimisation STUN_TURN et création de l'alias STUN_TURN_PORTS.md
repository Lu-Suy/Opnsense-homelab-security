**Projet :** Bastion Godmode / OPNsense Homelab Security  
**Date :** 31 juillet 2026  
**Lié au document :** 19 – Migration des règles Firewall OPNsense  
**Machine :** HP EliteDesk 800 G2 SFF (OPNsense 26.7.1_1)  

---

## 1. Contexte

Pendant et juste après la migration des règles firewall vers le nouveau système MVC/API, les logs montraient de nombreuses erreurs DNS (`NXDOMAIN`) sur les serveurs STUN/TURN.

Les noms suivants ne résolvaient plus :
- `stun.x.ai`
- `turn.x.ai`
- `stun.services.mozilla.com`

Ces entrées généraient du bruit inutile dans les logs et des tentatives de connexion vers des serveurs inexistants.

---

## 2. Objectifs de l’optimisation

1. Supprimer toutes les entrées mortes de l’alias `STUN_TURN`
2. Conserver uniquement des serveurs STUN/TURN publics fiables et performants
3. Créer un alias de ports propre (`STUN_TURN_PORTS`)
4. Remplacer les anciennes règles séparées (3478 et 5349) par **une seule règle** plus claire et maintenable
5. Ajouter le port **19302** (Google STUN) qui manquait

---

## 3. Contenu final de l’alias `STUN_TURN` (Hosts)

```
stun.l.google.com
stun1.l.google.com
stun2.l.google.com
stun3.l.google.com
stun4.l.google.com
stun.cloudflare.com
stun.stunprotocol.org
openrelay.metered.ca
```

**Serveurs retirés :**
- `stun.x.ai` / `turn.x.ai` → n’ont jamais existé en tant que serveurs publics
- `stun.services.mozilla.com` → décommissionné par Mozilla depuis plusieurs années

---

## 4. Création de l’alias `STUN_TURN_PORTS` (Port)

Ports inclus :
- **19302** → Google STUN
- **3478** → STUN/TURN classique
- **5349** → STUN/TURN over TLS

---

## 5. Nouvelle règle firewall (igc2_LAN)

| Paramètre              | Valeur                          |
|------------------------|---------------------------------|
| Action                 | Pass                            |
| Interface              | igc2_LAN                        |
| Protocol               | TCP/UDP                         |
| Source                 | igc2_LAN network                |
| Destination            | STUN_TURN                       |
| Destination Port       | STUN_TURN_PORTS                 |
| Description            | STUN/TURN optimisé (Grok Voice + WebRTC) |

Cette règle remplace les deux anciennes règles séparées (3478 et 5349).

La règle WebRTC Media Ports (UDP 50000-65535) a été conservée telle quelle.

---

## 6. Raisons techniques

- Réduction du bruit dans les logs DNS (plus de NXDOMAIN sur x.ai et Mozilla)
- Meilleure performance et fiabilité grâce aux serveurs Google + Cloudflare
- OpenRelay (Metered) conservé comme option TURN gratuite (avec limites)
- Une seule règle au lieu de plusieurs → plus simple à maintenir
- Ajout explicite du port 19302 qui était manquant

---

## 7. État final

- Alias `STUN_TURN` nettoyé et à jour
- Alias `STUN_TURN_PORTS` créé
- Une seule règle STUN/TURN propre et optimisée
- Règle WebRTC Media Ports toujours active
- Plus d’erreurs DNS liées aux anciens serveurs x.ai / Mozilla dans les logs




