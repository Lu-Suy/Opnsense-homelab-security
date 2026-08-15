# 11b – Configuration Port Forwarding sur FAI Box

**Date originale** : 22 mai 2026  
**Version portfolio** : 15 août 2026 (sanitisée + annotée)

> **⚠️ Annotation importante – Statut de cette fiche**  
>  
> Cette documentation décrit une **tentative de configuration** de port forwarding (FAI Box → OPNsense → BunkerWeb) réalisée en mai 2026.  
>  
> **Cette tentative n’a pas abouti de façon stable.**  
> Une erreur de configuration a été identifiée dans les règles NAT d’OPNsense :  
> - La règle HTTP redirigeait vers le port **8080** (port interne du conteneur)  
> - Alors qu’elle aurait dû rediriger vers le port **80** (port publié sur l’hôte, conformément au `docker-compose` de l’époque : `"80:8080"`).  
>  
> La configuration correcte aurait été :  
> - Port 80 → `10.0.10.x:80`  
> - Port 443 → `10.0.10.x:443`  
>  
> Cette correction a été identifiée, mais **elle n’a pas pu être appliquée** car la box FAI est tombée en panne peu de temps après et a dû être remplacée.  
>  
> **Cette approche a ensuite été abandonnée** au profit de **Cloudflare Tunnel** (beaucoup plus sécurisé et simple : plus besoin d’ouvrir les ports 80/443 sur la box ni de faire du double NAT).  
>  
> Conservée uniquement pour la **traçabilité historique** et la compréhension de l’évolution de l’architecture.

---

### Configuration Port Forwarding sur la box FAI

Dans l’interface de la box FAI, chercher les sections suivantes (elles peuvent avoir des noms légèrement différents) :

- **Port Forwarding**
- **Redirection de ports**
- **NAT**
- **Avancé → Redirections de ports**

**Ce qu’on devait créer (2 règles) :**

**Règle 1 – HTTP (port 80)**

- Nom : BunkerWeb_HTTP
- Protocole : **TCP**
- Port externe (ou Port public) : **80**
- Adresse IP interne (Destination) : IP WAN d’OPNsense
- Port interne : **80**

**Règle 2 – HTTPS (port 443)**

- Nom : BunkerWeb_HTTPS
- Protocole : **TCP**
- Port externe : **443**
- Adresse IP interne : IP WAN d’OPNsense
- Port interne : **443**

> ⚠️ **Important** : Si l’option "Activer UPnP" ou "DMZ" existe, **ne pas l’activer**. On fait des règles précises.

---

### Étape suivante : Créer les règles sur OPNsense

Aller dans OPNsense → **Firewall → NAT** (onglet **Port Forward**)

**1. Règle NAT pour HTTP (port 80 → 8080)**

Clique sur **+ Add** :

- **Interface** : WAN
- **Protocol** : TCP
- **Source** : any
- **Destination** : WAN address (ou This Firewall)
- **Destination port range** :
    - From : 80
    - To : 80
- **Redirect target IP** : `10.0.10.x` (IP services BunkerWeb)
- **Redirect target port** : 8080
- **Description** : NAT BunkerWeb HTTP (80->8080)
- Coche **Log**

![Capture d'écran](../images/Pasted%20image%2020260522122054.png)

### Que choisir ?

Pour notre cas, **choisir "Register rule"** (c’est le meilleur choix).

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
- **Redirect target IP** : `10.0.10.x`
- **Redirect target port** : 443
- **Description** : NAT BunkerWeb HTTPS
- Coche **Log**

Clique **Save** puis **Apply Changes** en haut.

![Capture d'écran](../images/Pasted%20image%2020260522122409.png)

Une fois les deux règles NAT sauvegardées et **Apply Changes** fait, aller dans :

![Capture d'écran](../images/Pasted%20image%2020260522122523.png)

**Firewall → Rules → WAN**

OPNsense a bien créé les deux nouvelles règles automatiquement :

![Capture d'écran](../images/Pasted%20image%2020260522122551.png)

---

### Situation à la date du document (22/05/2026)

- Box FAI → Redirection des ports 80 et 443 vers OPNsense
- OPNsense → Redirection des ports vers BunkerWeb (`10.0.10.x`)
- Règles Firewall WAN créées automatiquement

---

### Test final (très important)

1. **Appliquer les changements** si ce n’est pas encore fait (bouton **Apply Changes** en haut).
    
2. **Test externe** :
    
    - Depuis une machine hors du réseau local (ou téléphone en 4G/5G, **pas** en WiFi), ouvrir un navigateur en **navigation privée** et tester :
        
        - http://IP_PUBLIQUE
        - https://IP_PUBLIQUE (warning certificat normal à ce stade)
        - http://domaine.example (ou https)

---

**Document historique – Tentative de port forwarding (non finalisée)**  
*Version portfolio sanitisée et annotée – 15 août 2026*
