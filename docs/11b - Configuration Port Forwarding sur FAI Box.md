

### Configuration Port Forwarding sur Net Box.

Dans l’interface de ta box Nordnet, cherche les sections suivantes (elles peuvent avoir des noms légèrement différents) :

- **Port Forwarding**
- **Redirection de ports**
- **NAT**
- **Avancé → Redirections de ports**

**Ce qu’on doit créer (2 règles) :**

**Règle 1 – HTTP (port 80)**

- Nom : BunkerWeb_HTTP (ou Godmode_HTTP)
- Protocole : **TCP**
- Port externe (ou Port public) : **80**
- Adresse IP interne (Destination) : 192.168.5.244 (l’IP WAN de ton OPNsense)
- Port interne : **80**

**Règle 2 – HTTPS (port 443)**

- Nom : BunkerWeb_HTTPS
- Protocole : **TCP**
- Port externe : **443**
- Adresse IP interne : 192.168.5.244
- Port interne : **443**

> ⚠️ **Important** : Si tu as l’option "Activer UPnP" ou "DMZ", **ne l’active pas**. On fait des règles précises.




### Étape suivante : Créer les règles sur OPNsense

Va dans ton OPNsense → **Firewall → NAT** (onglet **Port Forward**)

**1. Règle NAT pour HTTP (port 80 → 8080)**

Clique sur **+ Add** :

- **Interface** : WAN
- **Protocol** : TCP
- **Source** : any
- **Destination** : WAN address (ou This Firewall)
- **Destination port range** :
    - From : 80
    - To : 80
- **Redirect target IP** : 10.0.10.10 (Prodesk_Web_Service)
- **Redirect target port** : 8080
- **Description** : NAT BunkerWeb HTTP (80->8080)
- Coche **Log**

![Capture d'écran](../images/Pasted%20image%2020260522122054.png)

### Que choisir ?

Pour notre cas, **choisis "Register rule"** (c’est le meilleur choix).

**Pourquoi ?**

- "Register rule" va automatiquement créer une règle Firewall sur l’onglet **WAN** qui autorise le trafic vers BunkerWeb.
- C’est plus propre et plus sûr que "Manual".


**2. Règle NAT pour HTTPS (port 443)**

Clique à nouveau sur **+ Add** :

- **Interface** : WAN
- **Protocol** : TCP
- **Source** : any
- **Destination** : WAN address
- **Destination port range** : 443 → 443
- **Redirect target IP** : 10.0.10.10
- **Redirect target port** : 443
- **Description** : NAT BunkerWeb HTTPS
- Coche **Log**

Clique **Save** puis **Apply Changes** en haut.


![Capture d'écran](../images/Pasted%20image%2020260522122409.png)


Une fois les deux règles NAT sauvegardées et **Apply Changes** fait, va dans :
![Capture d'écran](../images/Pasted%20image%2020260522122523.png)


**Firewall → Rules → WAN**
---OPNsense a bien créé les deux nouvelles règles automatiquement : 


![Capture d'écran](../images/Pasted%20image%2020260522122551.png)
**Excellent !** OPNsense a bien créé automatiquement les deux règles Firewall WAN. C’est nickel.

---

### Situation actuelle (22/05/2026)

- Nordnet Box → Redirection des ports 80 et 443 vers OPNsense (192.168.5.244)
- OPNsense → Redirection des ports vers BunkerWeb (10.0.10.10)
- Règles Firewall WAN créées automatiquement

---

### Test final (très important)

1. **Applique les changements** si ce n’est pas encore fait (bouton **Apply Changes** en haut).
    
2. **Test externe** :
    
    - Depuis AlphaDeck (ou un téléphone en 4G/5G, **pas** en WiFi), ouvre un navigateur en **navigation privée** et teste :
        
        - http://IP_PUBLIC
        - https://IP_PUBLIC (ça peut donner un warning certificat, c’est normal pour l’instant)
        - http://godmode.her (ou https)

**Dis-moi ce que ça donne.**

---

