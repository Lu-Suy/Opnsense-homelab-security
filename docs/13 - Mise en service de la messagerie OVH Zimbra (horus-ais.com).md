# 12a – Mise en service de la messagerie OVH Zimbra

**horus-ais.com – Réception & envoi fonctionnels**

**Date :** 28 juillet 2026  
**Projet :** Bastion Godmode / OPNsense Homelab Security  
**Lien avec le document parent :** [12 – Domaine & Messagerie externe](12-Domaine-et-Messagerie-externe-horus-ais.md)

---

## 1. Objectif

Rendre **opérationnelle** la messagerie `@horus-ais.com` hébergée chez OVH (Zimbra / Email Pro) afin de :

- Recevoir des e-mails sur le domaine
- Envoyer des e-mails (notamment les alertes du Bastion : OPNsense, Suricata, Fail2ban, BunkerWeb, monitoring…)
- Garantir une bonne délivrabilité (SPF + DKIM + DMARC)

Ce document décrit **exactement** les étapes réalisées et à réaliser, avec les valeurs DNS concrètes et les commandes de vérification.

---



## 2. Prérequis

| Élément | Statut | Détail |
|---------|--------|--------|
| Domaine | ✅ | `horus-ais.com` enregistré chez **Unstoppable Domains** |
| Service mail | ✅ | OVH **Zimbra / Email Pro** souscrit |
| Accès Unstoppable | ✅ | Panel DNS Records accessible |
| Accès OVH | ✅ | Espace client OVHcloud (Web Cloud → Zimbra / Email Pro) |
| Propagation | ⏳ | MX en place, attente de propagation complète (quelques minutes à 24 h) |

---



## 3. Étape 1 – Enregistrements MX (déjà réalisés le 28 juillet 2026)

### 3.1 Valeurs exactes configurées chez Unstoppable Domains

| Type  | Name                | Data             | Priority | TTL    |
| ----- | ------------------- | ---------------- | -------- | ------ |
| MX    | @                   | mx0.mail.ovh.net | **1**    | 1 hour |
| MX    | @                   | mx1.mail.ovh.net | **5**    | 1 hour |
| MX    | @                   | mx2.mail.ovh.net | **50**   | 1 hour |
| MX    | @                   | mx3.mail.ovh.net | **100**  | 1 hour |
| MX    | @                   | mx4.mail.ovh.net | **200**  | 1 hour |
| CNAME | ovh-zimbra-phhnty2j | ovh.com          | —        | 1 hour |

> Ces valeurs MX sont **exactement** celles recommandées par OVH pour Email Pro / Zimbra / MX Plan.  
> Le CNAME `ovh-zimbra-phhnty2j` est l’enregistrement de vérification fourni par OVH lors de l’association du domaine.


### 3.2 Commandes de vérification (à lancer depuis AlphaDeck ou n’importe quel Linux)

```bash
# Vérifier les enregistrements MX
dig MX horus-ais.com +short

# Alternative plus lisible
dig MX horus-ais.com

# Avec nslookup
nslookup -type=MX horus-ais.com
```

**Résultat attendu (après propagation) :**
```
1 mx0.mail.ovh.net.
5 mx1.mail.ovh.net.
50 mx2.mail.ovh.net.
100 mx3.mail.ovh.net.
200 mx4.mail.ovh.net.
```

**Explication des commandes :**
- `dig MX horus-ais.com +short` → interroge le DNS public et affiche uniquement les réponses MX (priorité + serveur).
- `+short` réduit le bruit pour une lecture rapide.
- Si rien n’apparaît ou si d’anciens MX apparaissent → la propagation n’est pas terminée.

### 3.3 Outils en ligne utiles
- https://mxtoolbox.com/SuperTool.aspx?action=mx%3ahorus-ais.com
- https://dnschecker.org/#MX/horus-ais.com

---



## 4. Étape 2 – Création des boîtes mail côté OVH

### 4.1 Boîtes prioritaires recommandées

| Adresse | Usage | Priorité |
|---------|-------|----------|
| `alertes@horus-ais.com` | Réception des alertes OPNsense / Suricata / Fail2ban / BunkerWeb | **Haute** |
| `contact@horus-ais.com` | Contact général / business | Haute |
| `ludovic@horus-ais.com` | Boîte personnelle principale | Haute |
| `admin@horus-ais.com` | Administration technique (optionnel) | Moyenne |


### 4.2 Procédure dans l’espace client OVH

1. Se connecter à [https://www.ovh.com/manager](https://www.ovh.com/manager)
2. Aller dans **Web Cloud** → **Zimbra** (ou **Email Pro** selon l’offre exacte)
3. Sélectionner le service associé à `horus-ais.com`
4. Onglet **Comptes e-mail** / **Email accounts**
5. Cliquer sur **Créer un compte** / **Add an account**
6. Remplir :
   - Adresse : `alertes` (le `@horus-ais.com` est ajouté automatiquement)
   - Mot de passe fort (minimum 12 caractères, majuscules, chiffres, symboles)
   - Quota : laisser par défaut (15 Go)
7. Valider
8. Répéter pour les autres boîtes

> **Important :** Note les mots de passe dans ton gestionnaire de mots de passe (Bitwarden / KeePassXC). Ne les stocke jamais en clair dans la documentation.


### 4.3 Accès Webmail

- URL classique OVH : `https://webmail.mail.ovh.net` ou l’URL indiquée dans ton espace client
- Identifiant = adresse e-mail complète (`alertes@horus-ais.com`)
- Mot de passe = celui défini à la création

![capture](../images/Pasted%20image%2020260728105157.png)
---



## 5. Étape 3 – Tests de réception

### 5.1 Test basique

1. Depuis une adresse externe (Gmail, Outlook, Proton, etc.) envoyer un e-mail à `alertes@horus-ais.com`
2. Attendre 1 à 5 minutes
3. Se connecter au webmail OVH et vérifier la réception
   
![capture](../images/Pasted%20image%2020260728105247.png)


### 5.2 Si le mail n’arrive pas

| Cause possible | Action |
|----------------|--------|
| Propagation MX pas terminée | Attendre + retester avec `dig MX` |
| Boîte pas encore créée | Vérifier dans l’espace client OVH |
| Filtre anti-spam OVH | Regarder le dossier Spam / Junk |
| Domaine pas correctement associé | Vérifier le diagnostic DNS dans l’espace client OVH |


### 5.3 Commande utile pour tracer

```bash
# Voir le chemin de livraison (si tu as accès aux logs, sinon via outils externes)
# Ou simplement utiliser :
dig MX horus-ais.com +trace
```

### 5.4 Envoi
- ✅ Le mail part bien.
- ⚠️ Il arrive actuellement **en Spam** chez Gmail.

**Cause principale :** domaine très récent → aucune réputation encore établie.


### 5.5 Authentification (Afficher l’original Gmail)

| Vérification | Résultat | Détail                       |
| ------------ | -------- | ---------------------------- |
| **SPF**      | **PASS** | IP 51.210.94.141 autorisée   |
| **DKIM**     | **PASS** | Sélecteur `ovhmo-selector-1` |
| **DMARC**    | **PASS** | Politique `p=none`           |
→ L’authentification technique est parfaitement correcte.




## 6. Étape 4 – SPF, DKIM et DMARC (à faire après réception confirmée)

Ces trois enregistrements sont **obligatoires** pour une bonne délivrabilité des alertes du Bastion. Sans eux, beaucoup de destinataires (Gmail, Outlook, etc.) classeront les mails en spam ou les rejetteront.
![capture](../images/Pasted%20image%2020260728114413.png)




### 6.1 SPF (Sender Policy Framework)

**But :** Autoriser uniquement les serveurs OVH à envoyer des e-mails au nom de `horus-ais.com`.

**Enregistrement à ajouter chez Unstoppable Domains :**

| Type | Name | Data | TTL |
|------|------|------|-----|
| TXT | @ | `v=spf1 include:mx.ovh.com ~all` | 1 hour |

**Explication :**
- `include:mx.ovh.com` → inclut tous les serveurs d’envoi OVH
- `~all` → softfail (recommandé au début). On pourra passer à `-all` plus tard une fois tout validé.


**Vérification :**
```bash
dig TXT horus-ais.com +short | grep spf
```


### 6.2 DKIM (DomainKeys Identified Mail)

**But :** Signer cryptographiquement chaque e-mail sortant pour prouver qu’il n’a pas été modifié et qu’il vient bien de ton service.

**Procédure :**
1. Dans l’espace client OVH → service Zimbra / Email Pro → **Informations générales** ou **Sécurité**
2. Activer DKIM (bouton souvent rouge au départ)
3. OVH te donnera **deux enregistrements CNAME** spécifiques à ton compte (ex. : `ovhempXXXX-selector1._domainkey` → quelque chose.dkim.mail.ovh.net)
4. Ajouter ces deux CNAME **exactement** chez Unstoppable Domains
5. Revenir dans OVH et cliquer sur **Activer / Enable**

> Les valeurs DKIM sont **personnelles** à ton service. Elles ne sont pas génériques. Tu dois les récupérer dans ton panel OVH.


Voici exactement ce que tu dois ajouter chez **Unstoppable Domains** :


### Enregistrement DKIM 1

|Champ|Valeur à mettre|
|---|---|
|**Type**|CNAME|
|**Name / Host**|ovhmo-selector-1._domainkey|
|**Value / Target**|ovhmo-selector-1._domainkey.4777530.jk.dkim.mail.ovh.net|
|**TTL**|1 hour (ou Automatic)|


### Enregistrement DKIM 2

|Champ|Valeur à mettre|
|---|---|
|**Type**|CNAME|
|**Name / Host**|ovhmo-selector-2._domainkey|
|**Value / Target**|ovhmo-selector-2._domainkey.4777531.jk.dkim.mail.ovh.net|
|**TTL**|1 hour (ou Automatic)|


### 6.3 DMARC (Domain-based Message Authentication)

**But :** Dire aux serveurs destinataires quoi faire si SPF ou DKIM échoue, et recevoir des rapports.

**Enregistrement recommandé (démarrage progressif) :**

| Type | Name | Data | TTL |
|------|------|------|-----|
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:alertes@horus-ais.com; pct=100;` | 1 hour |

**Explication des tags :**
- `p=none` → mode surveillance uniquement (ne rejette rien pour l’instant)
- `rua=mailto:alertes@horus-ais.com` → envoie les rapports agrégés vers ta boîte alertes
- Plus tard on pourra passer à `p=quarantine` puis `p=reject`

**Vérification :**
```bash
dig TXT _dmarc.horus-ais.com +short
```

---
![capture](../images/Pasted%20image%2020260728151435.png)





## 7. SRV (Autodiscover)

Le diagnostic OVH demande l’enregistrement suivant :

| Type | Name | Priority | Weight | Port | Target |
|------|------|----------|--------|------|--------|
| SRV | `_autodiscover._tcp` | 0 | 0 | 443 | `zimbra1.mail.ovh.net` |

**Problème :** Unstoppable Domains **ne supporte pas** le type d’enregistrement SRV.

**Décision :**  
Le SRV est laissé de côté pour le moment.  
Ce n’est **pas bloquant** pour l’envoi/réception.  
Il sert uniquement à l’autodiscover automatique des clients mail (Outlook, Thunderbird, etc.).

**Solution possible plus tard :** Changer les nameservers vers Cloudflare (gratuit) pour bénéficier du support SRV + plus de flexibilité DNS.




## 8. Checklist finale au 28 juillet 2026 (soir)

- [x] MX propagés et vérifiés
- [x] Boîte mail existante et fonctionnelle (`l.suy@horus-ais.com`)
- [x] Test de réception réussi
- [x] SPF configuré et **PASS**
- [x] DKIM configuré et **PASS**
- [x] DMARC configuré (`p=none`) et **PASS**
- [x] Test d’envoi réussi (mais arrive en Spam)
- [ ] SRV (non supporté par Unstoppable)
- [ ] Réputation du domaine à améliorer (temps + envois réguliers)
---



## 9. Paramètres clients mail (référence)

| Paramètre            | Valeur                                                                           |
| -------------------- | -------------------------------------------------------------------------------- |
| **IMAP (réception)** | `imap.mail.ovh.net` ou `ssl0.ovh.net` – Port **993** – SSL/TLS                   |
| **SMTP (envoi)**     | `smtp.mail.ovh.net` ou `ssl0.ovh.net` – Port **465** (SSL) ou **587** (STARTTLS) |
| **Identifiant**      | Adresse e-mail complète                                                          |
| **Mot de passe**     | Celui de la boîte                                                                |

---



## 10. Statut final au 28 juillet 2026 (soir)

| Élément | Statut | Commentaire |
|---------|--------|-------------|
| MX | ✅ | OK |
| Boîte mail | ✅ | `l.suy@horus-ais.com` Active |
| Réception | ✅ | Fonctionne |
| Envoi | ✅ / ⚠️ | Fonctionne mais tombe en Spam (domaine neuf) |
| SPF | ✅ | PASS |
| DKIM | ✅ | PASS |
| DMARC | ✅ | PASS (`p=none`) |
| SRV | ❌ | Non supporté par Unstoppable |
| Réputation domaine | ⏳ | À construire dans les prochains jours/semaines |
| Utilisation pro / candidatures | ⚠️ | Déconseillé pour le moment |

---

## 11. Prochaines actions recommandées

1. Continuer à envoyer quelques mails légitimes régulièrement pour « chauffer » le domaine.
2. Surveiller les rapports DMARC (quand ils arriveront).
3. Plus tard : envisager de passer le DMARC en `p=quarantine`.
4. Optionnel : migrer la zone DNS vers Cloudflare pour pouvoir ajouter le SRV et avoir plus de contrôle.
5. Pour les envois importants (employeurs, etc.) : continuer d’utiliser une adresse à forte réputation (Gmail) en attendant.

---

