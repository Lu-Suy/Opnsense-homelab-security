# 12 – Domaine & Messagerie externe

**horus-ais.com + OVH Zimbra Mail**

**Date originale :** 26 juillet 2026  
**Version portfolio :** 15 août 2026 (sanitisée + justifiée)

---

## 1. Pourquoi un nom de domaine et une messagerie professionnelle ?

Dans le cadre de la construction de l’architecture de défense, deux besoins concrets se sont imposés :

1. **Disposer d’une identité numérique propre et crédible**  
   Un nom de domaine dédié (`horus-ais.com`) permet de présenter les services (BunkerWeb, documentation, interfaces) de façon professionnelle, sans dépendre d’adresses IP brutes ou de sous-domaines génériques.

2. **Disposer d’un canal de notification fiable et professionnel**  
   Les alertes de sécurité (OPNsense, Suricata, Fail2ban, Wazuh, monitoring…) doivent pouvoir être envoyées depuis une adresse crédible (`@horus-ais.com`).  
   Une messagerie professionnelle améliore la délivrabilité et la perception des alertes par rapport à une boîte personnelle.

Ces deux éléments (domaine + messagerie) constituent donc des briques d’infrastructure à part entière, au service de la lisibilité et de l’opérabilité du lab.

---

## 2. Ce qui a été acquis

### 2.1 Domaine horus-ais.com

| Élément | Détail |
|---------|--------|
| **Fournisseur** | Unstoppable Domains |
| **Durée** | 3 ans |
| **Statut** | Enregistré |

Le domaine sert d’identité principale. Il n’héberge rien par lui-même. Il est pointé (DNS / records) vers les services exposés (BunkerWeb via Cloudflare Tunnel, etc.).

### 2.2 OVH Zimbra Mail (Email Pro)

| Élément | Détail |
|---------|--------|
| **Fournisseur** | OVHcloud |
| **Offre** | Zimbra / Email Pro |
| **Stockage** | 15 Go par boîte |
| **Engagement** | 12 mois |

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

## 3. Rôle dans l’architecture de défense

| Élément             | Rôle                                                                                   |
| ------------------- | -------------------------------------------------------------------------------------- |
| **horus-ais.com**   | Domaine principal des virtual hosts BunkerWeb + identité publique du projet            |
| **OVH Zimbra Mail** | Canal d’alertes sortantes (OPNsense, Suricata, Fail2ban, BunkerWeb, monitoring, Wazuh) |

L’utilisation d’adresses `@horus-ais.com` permet d’avoir des notifications professionnelles et crédibles provenant de l’infrastructure.

---

## 4. Prochaines étapes techniques (à la date du document)

1. Attendre la fin de la période de propagation / vérification OVH
2. Configurer les enregistrements MX du domaine pour pointer vers OVH Zimbra
3. Créer les premières boîtes (ex. : `contact@`, `alertes@`)
4. Préparer le virtual host `horus-ais.com` sur BunkerWeb
5. Configurer SPF / DKIM / DMARC pour la délivrabilité des alertes

---

## 5. Statut au 26 juillet 2026

- **Domaine :** Acheté et enregistré
- **OVH Zimbra :** Compte créé, service souscrit, en attente de configuration complète
- **Lien avec l’architecture :** Identifié et documenté

---

**Document de référence – Domaine et messagerie externe**  
*Version portfolio sanitisée – 15 août 2026*
