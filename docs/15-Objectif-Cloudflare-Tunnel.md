# 15 – Objectif du Cloudflare Tunnel

**Date originale :** juillet 2026  
**Version portfolio :** 15 août 2026 (sanitisée + enrichie)  
**Objectif :** Exposer de façon sécurisée le site servi par BunkerWeb, sans ouvrir de ports sur la box FAI ni sur OPNsense.

---

## Pourquoi Cloudflare Tunnel ?

L’objectif n’est pas seulement de rendre un site accessible depuis Internet.  
Il s’agit de le faire avec une **surface d’attaque drastiquement réduite**.

### Problèmes de l’approche classique (port-forwarding)

| Approche classique | Inconvénient |
|--------------------|--------------|
| Ouvrir les ports 80/443 sur la box FAI | Exposition directe de l’IP publique |
| NAT / Port Forward vers OPNsense puis vers BunkerWeb | Double NAT fragile, erreurs de configuration fréquentes |
| Let’s Encrypt en HTTP-01 direct | Nécessite que les ports soient ouverts et joignables |
| IP publique visible | Cible facile pour scans et attaques |

L’essai de mai 2026 (documents 11 + 11b) a montré les limites de cette méthode (erreurs de ports, box FAI instable, complexité).

### Avantages de Cloudflare Tunnel

| Avantage | Détail |
|----------|--------|
| **Aucun port ouvert localement** | Les ports 80/443 ne sont plus exposés sur la box ni sur OPNsense |
| **IP publique non exposée** | Le trafic arrive via le réseau Cloudflare, pas directement sur l’IP WAN |
| **TLS géré par Cloudflare** | Certificats et terminaison TLS côté Cloudflare |
| **Protection supplémentaire** | WAF, DDoS protection, bot management (même en plan Free) |
| **Simplicité opérationnelle** | Un seul composant (`cloudflared`) sur le bastion |
| **Fiabilité** | Solution mature, largement utilisée en production |

**En résumé :**  
Cloudflare Tunnel permet d’exposer des services web de façon moderne et sécurisée, en sortant du modèle « ouvrir des ports sur Internet ».

---

## 1. Situation du domaine `horus-ais.com`

Le domaine `horus-ais.com` était initialement côté **Unstoppable Domains**.

Deux options étaient possibles :

| Option | Avantage | Inconvénient |
|--------|----------|--------------|
| Garder Unstoppable + Cloudflare Tunnel | Pas de changement de registrar | Intégration plus limitée |
| Passer les nameservers vers Cloudflare | Meilleure intégration DNS, Tunnel, WAF, proxy | Gestion DNS déléguée à Cloudflare |

### Choix retenu

Les **nameservers** de `horus-ais.com` ont été basculés vers Cloudflare.

Le domaine reste la propriété du titulaire, mais la gestion DNS est confiée à Cloudflare.  
C’est la méthode la plus propre pour exploiter correctement Cloudflare Tunnel.

---

## 2. Unstoppable vs Cloudflare

| Critère | Unstoppable Domains | Cloudflare |
|---------|---------------------|------------|
| Philosophie | Web3 / privacy marketing | Infrastructure web mature |
| WHOIS | Moins classique | Classique, masquable |
| Maturité | Plus jeune | Très éprouvé en production |
| Contrôle DNS | Plus limité (pas de SRV, moins de flexibilité) | Excellent |
| Sécurité / Tunnel | Pas natif | Excellent (Cloudflare Tunnel + WAF) |

**En résumé :**
- Unstoppable est plus « privacy-oriented » sur le papier (Web3).
- Cloudflare est **beaucoup plus puissant et fiable** pour ce qu’on veut faire (Tunnel, protection, DNS avancé).  
  Beaucoup de professionnels en cybersécurité l’utilisent justement parce qu’il est robuste.

---

## 3. Mise en place Cloudflare

### Étape 1 – Compte Cloudflare

Création / connexion au compte Cloudflare :  
[https://dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)

### Étape 2 – Ajout du domaine

1. Dans le dashboard Cloudflare : **Add a domain** / **Add site**
2. Saisir :
   ```text
   horus-ais.com
   ```
3. Continuer
4. Choisir le plan **Free**
5. Laisser Cloudflare scanner les DNS existants, puis confirmer

![Capture ajout domaine Cloudflare](../images/Pasted%20image%2020260730141503.png)

---

## 4. Changement des nameservers chez Unstoppable

Une fois les nameservers Cloudflare affichés, la bascule se fait côté Unstoppable :

1. Ouvrir le menu **Nameservers**
2. Passer en mode **Custom**
3. Renseigner les nameservers Cloudflare, par exemple :
   ```text
   armando.ns.cloudflare.com
   sonia.ns.cloudflare.com
   ```
4. Enregistrer

![Capture nameservers Unstoppable](../images/Pasted%20image%2020260730141644.png)

### Point important

Les records NS visibles dans “DNS Records” ne sont pas la bonne zone de modification.  
Le changement se fait bien dans le paramètre **Nameservers** du domaine.

![Capture confirmation nameservers](../images/Pasted%20image%2020260730142123.png)

Après bascule en custom nameservers, Unstoppable indique en général :

> DNS management is unavailable when custom nameservers are set

C’est le comportement attendu :
- Unstoppable ne gère plus le DNS
- toute la gestion DNS passe par Cloudflare

---

## 5. Validation du statut Cloudflare

Après propagation, le domaine doit passer de **Pending** à **Active** dans Cloudflare.

Le délai peut aller de quelques minutes à plusieurs heures selon la propagation DNS.

![Capture statut Active](../images/Pasted%20image%2020260730142638.png)

Dans ce cas, le domaine `horus-ais.com` est passé rapidement en **Active**.

---

## 6. Suite logique

Une fois le DNS opérationnel chez Cloudflare, l’étape suivante consiste à installer et configurer `cloudflared` sur le bastion, afin de créer le tunnel vers BunkerWeb.

Cette partie est documentée dans :

- [16 – Installer cloudflared et Cloudflare Tunnel](./16%20-%20Installer%20cloudflared%20et%20Cloudflare%20Tunnel%20(Prodesk).md)
- [20 – Hardening cloudflared – utilisateur dédié non-root](./20%20-%20Hardening%20cloudflared%20-%20utilisateur%20dédié%20non-root.md)

---

## Résultat attendu

À ce stade :

- le domaine est géré côté Cloudflare
- l’architecture est prête pour une exposition HTTPS **sans ouvrir de ports locaux**
- la suite consiste à brancher le tunnel sur BunkerWeb

---

**Dernière mise à jour originale :** août 2026  
**Statut :** DNS Cloudflare opérationnel – tunnel documenté dans les fiches 16 et 20

**Document de référence – Objectif et justification du Cloudflare Tunnel**  
*Version portfolio sanitisée et enrichie – 15 août 2026*
