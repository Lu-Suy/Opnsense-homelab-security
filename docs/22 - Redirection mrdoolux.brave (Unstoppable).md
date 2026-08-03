
### Ce que tu as déjà

Tu as un **site Web2 classique** qui fonctionne :
```
https://mrdoolux.horus-ais.com
```
N’importe qui sur Internet peut y accéder en tapant cette adresse (navigateur normal, téléphone, etc.).

### Ce qu’est `mrdoolux.brave`

C’est un **domaine Web3** (blockchain).  
Il ne se comporte **pas** comme un `.com` :

- Il n’est **pas** résolu par le DNS classique de la majorité des navigateurs.
- Il est visible principalement dans **Brave**, dans certains wallets, et via les resolvers Unstoppable.
- Les gens qui évoluent dans l’écosystème Web3 / crypto le connaissent et peuvent le taper directement.

### À quoi sert la redirection alors ?

La redirection sert à **relier les deux mondes** :

| Situation | Sans redirection | Avec redirection |
|-----------|------------------|------------------|
| Quelqu’un tape `mrdoolux.brave` (dans Brave ou via Unstoppable) | Il arrive sur la page par défaut Unstoppable / IPFS / profil | Il arrive sur **ton vrai site** `mrdoolux.horus-ais.com` |
| Quelqu’un tape `mrdoolux.horus-ais.com` | Il arrive sur ton site | Pareil |

**En résumé :**
- `mrdoolux.horus-ais.com` = ton site pro, accessible à tout le monde.
- `mrdoolux.brave` = ton **identité / branding Web3**.  
  La redirection permet que les gens qui te connaissent via le nom Web3 arrivent quand même sur ton vrai contenu.

C’est utile si tu veux :
- Garder une présence dans l’écosystème Brave / crypto
- Utiliser le nom court et original `.brave`
- Montrer que tu maîtrises les deux couches (Web2 + Web3)

Ce n’est **pas obligatoire**.  
Si tu n’as pas d’intérêt particulier pour le branding Web3, on peut très bien laisser `mrdoolux.brave` de côté (ou le mettre en IPFS pure plus tard).

---


### On attaque Unstoppable

1. Connecte-toi sur [https://unstoppabledomains.com](https://unstoppabledomains.com)
2. Va dans **My Domains**
3. Clique sur **mrdoolux.brave**
4. Dans le menu de gauche, cherche **Website** (ou **Manage** → Website / Redirect)

### Pour faire une redirection vers ton site Web2

D’après l’interface et la doc Unstoppable, il faut généralement :

1. **Supprimer** le site IPFS actuel (bouton rouge **Remove**)
2. Ensuite configurer la redirection vers https://mrdoolux.horus-ais.com

**Méthode la plus courante chez Unstoppable :**

1. Clique sur **Remove** (ça va te demander de signer une transaction avec ton wallet)
2. Une fois retiré, tu auras la possibilité d’uploader un petit fichier HTML de redirection ou d’utiliser leur option de redirect.


![capture](../images/Pasted%20image%2020260801181557.png)


L’interface ne prend **que** un hash IPFS.  
Pour rediriger vers un site Web2 classique, Unstoppable demande de passer par leur guide (créer un petit fichier HTML de redirection et l’uploader sur IPFS).

### Méthode à suivre

On va créer un fichier HTML très simple qui redirige automatiquement vers ton site.

**1. Crée le fichier sur ton PC**

Ouvre le Bloc-notes et colle exactement ceci :

```html
<!DOCTYPE html>
<html>
<head>
    <title>Redirecting...</title>
    <meta http-equiv="refresh" content="0; url=https://mrdoolux.horus-ais.com" />
    <script>
        window.location.href = "https://mrdoolux.horus-ais.com";
    </script>
</head>
<body>
    <p>Redirection vers <a href="https://mrdoolux.horus-ais.com">mrdoolux.horus-ais.com</a>...</p>
</body>
</html>
```

**2. Enregistre-le** sous le nom :
```
index.html
```
(Attention à bien mettre l’extension `.html` et pas `.txt`)

**3. Retourne sur Unstoppable**
- Reviens à la page Website de `mrdoolux.brave`
- Clique sur **Upload Website Files**
- Envoie ton fichier `index.html`
- Clique sur **Launch Website**
- **Signe la transaction**

Une fois la transaction confirmée, `mrdoolux.brave` redirigera vers `https://mrdoolux.horus-ais.com`.

---

Tu veux que je te redonne le code HTML plus clairement, ou tu l’as déjà ?  
Dis-moi quand tu as uploadé et signé.
