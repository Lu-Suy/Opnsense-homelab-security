# 04b – VLANs et Segmentation Réseau

**Date originale** : 04 juillet 2026  
**Version portfolio** : 15 août 2026 (sanitization)  
**Objectif** : Isoler les zones réseau pour sécuriser l’exposition publique et préparer une zone lab isolée.

> **Note portfolio** :  
> Les adresses IP et hostnames ont été généralisés.  
> Contenu technique et procédure conservés intégralement.

---

## Philosophie appliquée

- Default Deny partout
- Segmentation stricte (Bastion / Zone lab / LAN clients)
- Règles explicites + logging
- Prévention du pivot en cas de compromission

---

## Topologie proposée

- **LAN** (igc2 - `10.0.0.0/24`) → Machines de confiance (workstation admin, etc.)
- **OPT1 Bastion** (igc1 - `10.0.10.0/24`) → Bastion + BunkerWeb (zone exposée)
- **OPT2 Lab** (igc0 - `10.0.20.0/24`) → Zone très isolée pour machine lab / futurs tests

---

## Règles recommandées par interface (à créer)

### OPT1 - Bastion (`10.0.10.0/24`)
- Sortie Internet limitée (DNS, NTP, HTTP/HTTPS)
- Entrée uniquement depuis LAN (workstation admin) pour admin
- Block All en dernier

### OPT2 - Lab (`10.0.20.0/24`)
- Très restrictif
- Sortie Internet autorisée (mises à jour)
- Aucune communication vers LAN ou OPT1 sauf exception SSH depuis workstation admin
- Block All en dernier

---

## Prochaines étapes (à l’époque)

1. Créer les VLANs sur OPNsense (Interfaces → Assign)
2. Configurer les interfaces OPT1 et OPT2
3. Mettre en place les règles firewall correspondantes
4. Tester l’isolation (ping, etc.)

**Statut à la date du document** : À faire

---

### Avant de commencer

- Connecte-toi sur l’interface OPNsense.
- Fais une sauvegarde de la config actuelle : **System → Configuration → Backups** → Download.

---

### Création des VLANs (étape par étape)

**1. Va dans Interfaces → Devices → VLAN**

Clique sur le bouton **+ Add**

**Pour OPT1 - Bastion (déjà existant mais on vérifie / recrée si besoin)**

- **Parent Interface** : igc1
- **VLAN Tag** : 10
- **Description** : OPT1 - Bastion (10.0.10.0/24)
- **Save**

**Pour OPT2 - Lab (nouveau)**

- **Parent Interface** : igc0
- **VLAN Tag** : 20
- **Description** : OPT2 - Zone Isolée (10.0.20.0/24)
- **Save**

Clique **Apply Changes** en haut.

![Capture d’écran](../images/Pasted%20image%2020260704234656.png)

---

**2. Assignation des interfaces**

Va dans **Interfaces → Assignments**

- Cherche l’interface OPT1 → assigne-la à igc1.10 (ou le nom du VLAN que tu as créé)
- Cherche l’interface OPT2 → assigne-la à igc0.20

Puis pour chaque interface :

**Pour OPT1 (Bastion)** :

- Enable
- IPv4 Configuration Type : Static IPv4
- IPv4 Address : `10.0.10.1 / 24`
- Description : Bastion

**Pour OPT2 (Lab)** :

- Enable
- IPv4 Configuration Type : Static IPv4
- IPv4 Address : `10.0.20.1 / 24`
- Description : Zone Isolée / Lab

**Apply Changes**

![Capture d’écran](../images/Pasted%20image%2020260704234602.png)

### Ce qu’on vient de faire concrètement :

- On a créé **deux réseaux virtuels séparés** sur la même carte réseau physique d’OPNsense.
- OPT1 (VLAN 10) → Zone Bastion
- OPT2 (VLAN 20) → Zone lab isolée

Maintenant les deux zones ont leur propre sous-réseau :
- OPT1 : `10.0.10.0/24`
- OPT2 : `10.0.20.0/24`

C’est le premier pas de la **segmentation** que tu voulais.

---

### Prochaines étapes pour tester que ça fonctionne

**1. Apply Changes**

Clique sur le bouton **Apply Changes** (en haut ou en bas) pour valider les IP.

**2. Tests de base (à faire depuis la workstation admin ou une machine sur LAN)**

```bash
# Test accès à OPT1 (Bastion)
ping 10.0.10.1
ping 10.0.10.x   # IP services du bastion

# Test accès à OPT2 (Lab)
ping 10.0.20.1
```

**3. Test d’isolation (important)**

Depuis une machine sur **LAN** (`10.0.0.x`) :
- Tu dois pouvoir joindre OPT1 (`10.0.10.x`)
- Tu **ne dois pas** pouvoir joindre OPT2 (`10.0.20.x`) pour l’instant (on le configurera plus tard si besoin)

---

**Document historique – Segmentation VLAN – 04 juillet 2026**  
*Version portfolio sanitisée – 15 août 2026*
