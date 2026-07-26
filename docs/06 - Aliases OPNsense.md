

## Aliases créés par nous

| Alias               | Type       | Contenu (une par ligne)                                                                         | Description                             |
| ------------------- | ---------- | ----------------------------------------------------------------------------------------------- | --------------------------------------- |
| Alpha_4_Deck        | Host       | 10.0.0.10/24                                                                                    | Admin_Opnsense                          |
| Prodesk_Manager     | Hosts      | 10.0.10.11                                                                                      | Règles SSH Prodesk                      |
| Prodesk_web_service | Hots       | 10.0.10.10                                                                                      | Règle WEB Services                      |
| **GX10**            | Hosts      | 10.0.20.10                                                                                      | Règles OPT2 (machine haute performance) |
| **DNS_External**    | Host(s)    | 1.1.1.1 1.0.0.1 9.9.9.9                                                                         | DNS externes fiables                    |
| **NTP_Servers**     | Host(s)    | 0.pool.ntp.org 1.pool.ntp.org 2.pool.ntp.org 3.pool.ntp.org time.cloudflare.com time.google.com | Serveurs NTP                            |
| **STUN_TURN**       | Host(s)    | stun.x.ai turn.x.ai stun.l.google.com stun1.l.google.com stun2.l.google.com                     | STUN/TURN pour Grok Voice & WebRTC      |
| **RFC1918**         | Network(s) | 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16                                                         | Réseaux privés (utile pour bloquer)     |

## Aliases système automatiques (ne pas modifier)

| Nom            | Type                  | Rôle                                                                 | À toucher ? |
|----------------|-----------------------|----------------------------------------------------------------------|-------------|
| bogons         | External (advanced)   | Réseaux invalides / spoofing (IPv4)                                 | Jamais     |
| bogonsv6       | External (advanced)   | Réseaux invalides / spoofing (IPv6)                                 | Jamais     |
| sshlockout     | External (advanced)   | Protection anti-bruteforce SSH                                      | Jamais     |
| virusprot      | External (advanced)   | Liste d’IP malveillantes / botnets                                  | Jamais     |
| __wan_network  | Internal              | wan net                                                              | Automatique|
| __opt1_network | Internal              | opt1 net                                                             | Automatique|
| __opt2_network | Internal              | opt2 net                                                             | Automatique|
| __lan_network  | Internal              | lan net                                                              | Automatique|

**Dernière mise à jour** :  05/ 07/ 2026