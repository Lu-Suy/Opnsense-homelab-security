# 22 – Redirection mrdoolux.brave (Unstoppable)

**Date originale :** début août 2026  
**Version portfolio :** 15 août 2026 (sanitisée)  
**Objectif :** Relier l’identité Web3 `mrdoolux.brave` au site Web2 réel `https://mrdoolux.horus-ais.com`.

---

## Pourquoi cette redirection ?

On a un **site Web2 classique** qui fonctionne :

```
https://mrdoolux.horus-ais.com
```

N’importe qui sur Internet peut y accéder.

### Ce qu’est `mrdoolux.brave`

C’est un **domaine Web3** (blockchain).  
Il ne se comporte **pas** comme un `.com` :

- Il n’est **pas** résolu par le DNS classique de la majorité des navigateurs.
- Il est visible principalement dans **Brave**, dans certains wallets, et via les resolvers Unstoppable.
- Les personnes qui évoluent dans l’écosystème Web3 / crypto le connaissent et peuvent le taper directement.

### À quoi sert la redirection ?

La redirection sert à **relier les deux mondes** :

| Situation | Sans redirection | Avec redirection |
|-----------|------------------|------------------|
| Quelqu’un tape `mrdoolux.brave` (dans Brave ou via Unstoppable) | Il arrive sur la page par défaut Unstoppable / IPFS / profil | Il arrive sur **le vrai site** `mrdoolux.horus-ais.com` |
| Quelqu’un tape `mrdoolux.horus-ais.com` | Il arrive sur le site | Pareil |

**En résumé :**
- `mrdoolux.horus-ais.com` = site pro, accessible à tout le monde.
- `mrdoolux.brave` = identité / branding Web3.  
  La redirection permet que les gens qui connaissent le nom Web3 arrivent quand même sur le vrai contenu.

C’est utile si on veut :
- Garder une présence dans l’écosystème Brave / crypto
- Utiliser le nom court et original `.brave`
- Montrer qu’on maîtrise les deux couches (Web2 + Web3)

Ce n’est **pas obligatoire**.  
Si le branding Web3 n’est pas prioritaire, on peut laisser `mrdoolux.brave` de côté (ou le mettre en IPFS pure plus tard).

---

## À faire sur Unstoppable

1. Se connecter sur [https://unstoppabledomains.com](https://unstoppabledomains.com)
2. Aller dans **My Domains**
3. Cliquer sur **mrdoolux.brave**
4. Dans le menu de gauche, chercher **Website** (ou **Manage** → Website / Redirect)

### Pour faire une redirection vers le site Web2

1. **Supprimer** le site IPFS actuel (bouton rouge **Remove**)
2. Ensuite configurer la redirection vers `https://mrdoolux.horus-ais.com`

**Méthode la plus courante chez Unstoppable :**

1. Cliquer sur **Remove** (signature de transaction avec le wallet)
2. Une fois retiré, uploader un petit fichier HTML de redirection ou utiliser leur option de redirect.

![capture](../images/Pasted%20image%2020260801181557.png)

L’interface ne prend **que** un hash IPFS.  
Pour rediriger vers un site Web2 classique, Unstoppable demande de passer par un fichier HTML de redirection uploadé sur IPFS.

---

## Méthode à suivre

### 1. Créer le fichier de redirection

Ouvrir un éditeur de texte et coller exactement ceci :

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

### 2. Enregistrer le fichier

Sous le nom :

```
index.html
```

(Attention à bien mettre l’extension `.html` et non `.txt`)

### 3. Retourner sur Unstoppable

- Revenir à la page Website de `mrdoolux.brave`
- Cliquer sur **Upload Website Files**
- Envoyer le fichier `index.html`
- Cliquer sur **Launch Website**
- **Signer la transaction**

Une fois la transaction confirmée, `mrdoolux.brave` redirigera vers `https://mrdoolux.horus-ais.com`.

---

**Document de référence – Redirection Web3 → Web2**  
*Version portfolio sanitisée – 15 août 2026*
