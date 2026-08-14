

### Objectif du Cloudflare Tunnel

Permettre d’accéder de l’extérieur (et de te montrer) le site servi par BunkerWeb sur la Prodesk (10.0.10.10) de façon sécurisée, sans port forwarding classique.


### 1. Est-ce que `horus-ais.com` est encore chez Unstoppable ?
Oui, d’après tout ce qu’on a documenté jusqu’ici, le domaine est toujours enregistré chez **Unstoppable Domains**.

### 2. Unstoppable vs Cloudflare : lequel est plus « confidentiel » ?

| Critère                    | Unstoppable Domains                  | Cloudflare                          |
|---------------------------|--------------------------------------|-------------------------------------|
| **Philosophie**           | Web3 / blockchain, marketing privacy | Grande entreprise US, très utilisée |
| **WHOIS**                 | Moins classique, orientation privacy | WHOIS classique (mais on peut le masquer) |
| **Confiance / maturité**  | Plus jeune, moins éprouvé pour de l’infra sérieuse | Très mature, énormément utilisé en production |
| **Contrôle DNS**          | Limité (pas de SRV, moins de flexibilité) | Excellent |
| **Sécurité / Tunnel**     | Pas natif                            | Excellent (Cloudflare Tunnel + WAF) |

**En résumé clair :**
- Unstoppable est plus « privacy-oriented » sur le papier (Web3).
- Cloudflare est **beaucoup plus puissant et fiable** pour ce qu’on veut faire (Tunnel, protection, DNS avancé). Beaucoup de gens sérieux en cybersécurité l’utilisent justement parce qu’il est robuste.

### 3. Est-ce que je suis obligé de le mettre chez Cloudflare ?

**Non, ce n’est pas obligatoire**, mais c’est **fortement recommandé**.

Deux possibilités :

| Option | Avantage | Inconvénient |
|--------|----------|--------------|
| **Garder chez Unstoppable** + Cloudflare Tunnel | Tu ne changes pas de registrar | Plus limité, configuration un peu plus bricolée |
| **Déplacer les nameservers vers Cloudflare** (recommandé) | Meilleure intégration, plus simple, plus de puissance (Proxy orange, WAF, analytics, etc.) | Tu « déclares » le domaine chez Cloudflare |

**Ma recommandation pour le projet Bastion Godmode :**
On passe les **nameservers** de `horus-ais.com` chez Cloudflare.  
Tu gardes le domaine acheté chez Unstoppable (tu restes propriétaire), mais tu confies la gestion DNS à Cloudflare. C’est la méthode standard et la plus propre.

---

### Étape 1 – Compte Cloudflare

1. Est-ce que tu as **déjà un compte Cloudflare** ?
2. Si non, crée-en un ici : [https://dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)

Une fois que tu es connecté (ou que tu as créé le compte), dis-moi :

> « Je suis connecté à Cloudflare »

Ensuite on passera à l’ajout du domaine `horus-ais.com` et au changement des nameservers chez Unstoppable.

### Étape 2 – Ajouter le domaine `horus-ais.com` dans Cloudflare

1. Dans le dashboard Cloudflare, clique sur **“Add a domain”** ou **“Onboard a domain”** (ou “Add site”).
2. Tape exactement :  
   ```
   horus-ais.com
   ```
3. Clique sur **Continue**.
4. Choisis le plan **Free** (c’est largement suffisant pour ce qu’on fait).
5. Cloudflare va scanner les DNS existants. Laisse-le faire, puis clique sur **Continue** / **Confirm DNS records**.


![capture](../images/Pasted%20image%2020260730141503.png)


Une fois arrivé à l’écran qui te montre les **deux nameservers Cloudflare** (du style `xxxx.ns.cloudflare.com`),

### Ce qu’il faut faire :

Regarde dans le menu de gauche de ta capture :

Tu as clairement l’entrée **Nameservers** (juste en dessous de DNS Records).

1. Clique sur **Nameservers** (dans le menu de gauche).
2. Là tu auras le choix entre les nameservers Unstoppable (par défaut) et **Custom**.
3. Passe en **Custom**.
4. Mets les deux de Cloudflare :

```
armando.ns.cloudflare.com
sonia.ns.cloudflare.com
```

![capture](../images/Pasted%20image%2020260730141644.png)

5. Sauvegarde.

Les records NS que tu vois dans « DNS Records » sont juste l’affichage des nameservers actuels.  
On ne les modifie **pas** en ajoutant de nouveaux records NS. On change le paramètre **Nameservers** du domaine.

![capture](../images/Pasted%20image%2020260730142123.png)


**C’est parfaitement normal et attendu.**

Dès que tu passes en **Custom Nameservers**, Unstoppable te dit :

> « DNS management is unavailable when custom nameservers are set »

C’est **exactement** ce qu’on voulait.

À partir de maintenant :
- Unstoppable ne gère plus les DNS de `horus-ais.com`
- Toute la gestion DNS se fait **uniquement dans Cloudflare**

C’est le comportement correct.

### Prochaine étape

Va dans ton dashboard Cloudflare et regarde le statut du domaine `horus-ais.com`.

Il doit être encore en **Pending** (en attente).  
Ça peut prendre de quelques minutes à plusieurs heures (parfois jusqu’à 24h, mais souvent beaucoup plus rapide).

Quand le statut passe à **Active**, Cloudflare t’envoie généralement un email, et tu verras le statut changer dans le dashboard.

Observe ce que tu vois actuellement comme statut sur Cloudflare (Pending / Active / autre).

![capture](../images/Pasted%20image%2020260730142638.png)
**C’est parfait !**

Le domaine `horus-ais.com` est déjà **Active**.  
Cloudflare a détecté le changement de nameservers très rapidement (ça arrive souvent quand la propagation est bonne).



### Prochaine étape : installer Cloudflare Tunnel sur la Prodesk

On va maintenant installer `cloudflared` sur la Prodesk pour créer le tunnel qui va exposer BunkerWeb.


