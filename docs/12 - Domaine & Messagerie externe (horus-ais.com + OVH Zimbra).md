# 12 – Domaine & Messagerie externe

**horus-ais.com + OVH Zimbra Mail**

**Date :** 26 juillet 2026  
**Projet :** Bastion Godmode / OPNsense Homelab Security

---

## 1. Contexte d’origine – Pourquoi ces achats ?

Avant de basculer pleinement sur le projet cybersécurité (Bastion Godmode / OPNsense), plusieurs pistes business avaient été explorées :

Dans ce contexte, le nom de domaine **horus-ais.com** a été choisi comme identité numérique centrale, et un service de messagerie professionnelle a été souscrit chez OVH pour avoir des adresses crédibles (`@horus-ais.com`).

---

## 2. Ce qui a été acheté concrètement

### 2.1 Domaine horus-ais.com

| Élément | Détail |
|---------|--------|
| **Fournisseur** | Unstoppable Domains |
| **Durée** | 3 ans |
| **Coût approximatif** | ~30 $ (promo) |
| **Statut** | Enregistré – Verrouillage de transfert 60 jours en cours |

Le domaine sert d’identité principale. Il n’héberge rien par lui-même. Il doit être pointé (DNS / A records ou nameservers) vers un service (BunkerWeb sur le Prodesk, ou autre).

### 2.2 OVH Zimbra Mail (Email Pro)

| Élément | Détail |
|---------|--------|
| **Fournisseur** | OVHcloud |
| **Offre** | Zimbra / Email Pro |
| **Stockage** | 15 Go par boîte |
| **Engagement** | 12 mois |
| **Coût** | environ 1,59 € HT / mois / boîte (≈ 22,90 € TTC / an pour 1 boîte) |

**Ce que cette offre permet :**
- Créer des adresses professionnelles (`@horus-ais.com`)
- Webmail Zimbra
- IMAP / POP / SMTP
- Anti-spam et filtrage de base

**Ce que cette offre ne permet PAS :**
- Héberger un site web
- Avoir un serveur web ou un reverse-proxy
- Stockage de fichiers pour un site

---

## 3. Utilisation actuelle dans Bastion Godmode

Même si l’origine était business (High-Ticket / Majordome), ces deux briques sont maintenant intégrées dans l’architecture de défense :

| Élément | Rôle dans le Bastion |
|---------|----------------------|
| **horus-ais.com** | Futur virtual host sur BunkerWeb (Let’s Encrypt) + documentation / interface d’administration |
| **OVH Zimbra Mail** | Canal d’alertes sortantes (OPNsense, Suricata, Fail2ban, BunkerWeb, monitoring) |

---

## 4. Clarification importante

Les idées « High-Ticket Agency » et « Majordome High-Tech » n’ont pas disparu. Elles restent des pistes business possibles. Cependant, le domaine et les mails ont été achetés dans un moment où ces pistes étaient actives, et ils sont aujourd’hui réutilisés de façon pragmatique pour le projet technique Bastion Godmode.

Il n’y a pas de contradiction : un même domaine peut servir à la fois d’identité business et d’infrastructure technique.

---

## 5. Prochaines étapes techniques

1. Attendre la fin de la période de propagation / vérification OVH (jusqu’à 48 h)
2. Configurer les enregistrements MX du domaine pour pointer vers OVH Zimbra
3. Créer les premières boîtes (ex. : `contact@`, `alertes@`, `ludovic@`)
4. Préparer le virtual host `horus-ais.com` sur BunkerWeb (fichier 11 de la doc)
5. Configurer SPF / DKIM / DMARC pour la délivrabilité des alertes

---

## 6. Statut au 26 juillet 2026

- **Domaine :** Acheté et enregistré (Unstoppable)
- **OVH Zimbra :** Compte créé, service souscrit, en attente de configuration complète
- **Lien avec Bastion :** Identifié et documenté
- **Documentation :** Ce fichier 

---

