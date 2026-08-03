

**Projet :** Bastion Godmode / OPNsense Homelab Security  
**Date :** 31 juillet 2026  
**Version OPNsense :** 26.7.1_1 (amd64)  
**Machine :** HP EliteDesk 800 G2 SFF (Firewall)  
**Auteur :** Documentation technique lab  

---

## 1. Contexte

Lors de la mise à jour vers OPNsense **26.7.1_1**, le système a basculé les règles firewall vers le nouveau moteur **MVC/API**.  

L’ancien système de règles (pages PHP statiques) est devenu « legacy ». Un **Migration Assistant** est apparu pour accompagner le passage.

### Raisons techniques (sourcées)

- **OPNsense 26.1** (« Witty Woodpecker ») : introduction d’une nouvelle interface de règles firewall basée sur MVC/API. L’ancien système reste disponible en parallèle.
- **OPNsense 26.7** (« Xenial Xenops ») : 
  - Les règles firewall passent par défaut sur le nouveau système MVC/API.
  - Les pages legacy sont déplacées dans le plugin `os-firewall-legacy`.
  - Un Migration Assistant est fourni pour migrer proprement les règles existantes.

Sources officielles :
- Release notes OPNsense 26.7
- Release notes OPNsense 26.1
- Documentation officielle + commits du core (déplacement vers le plugin legacy)

---

## 2. État du système avant migration

- Fichier système : **UFS** (pas de ZFS) → `zfs list` retourne `no datasets available`
- Pas de snapshots possibles
- Méthode de sauvegarde retenue : **System → Configuration → Backups** (fichier XML) + Configuration History
- Anti-lockout : laissé **activé** (case « Disable anti-lockout » non cochée)

### ZFS vs UFS – Explication simple

OPNsense (comme FreeBSD) propose deux grands modes d’installation pour le système de fichiers :

|Mode|Nom complet|Avantages|Inconvénients|Notre cas|
|---|---|---|---|---|
|**UFS**|Unix File System|Simple, léger, très stable|Pas de snapshots natifs|**Oui**|
|**ZFS**|Zettabyte File System|Snapshots, checksums, compression, clones|Plus lourd en RAM, un peu plus complexe|Non|


## 3. Processus de migration réalisé (31 juillet 2026)

### Étape 1 – Sauvegarde
- Téléchargement de la configuration complète via **System → Configuration → Backups** (fichier XML)
- Vérification que l’anti-lockout reste activé

### Étape 2 – Export des règles legacy
- Ouverture du **Migration Assistant**
- Clic sur **Export current rules**
- Téléchargement du fichier CSV (ex. : `download_rules(2).csv`)

### Étape 3 – Import dans le nouveau système
- Dans **Firewall → Rules** (nouvelle interface)
- Utilisation du bouton d’import (icône upload)
- Sélection du fichier CSV exporté
- Validation de l’import
- Clic sur **Apply**

### Étape 4 – Vérifications post-import
Contrôles effectués avec succès :
- Accès à l’interface web OPNsense : OK
- SSH depuis AlphaDeck (`10.0.0.10`) vers `10.0.0.1` : OK
- Accès BunkerWeb (ports 80/443) : OK
- Règles anti-lockout toujours présentes
- Règles critiques (DNS, NTP, HTTPS outbound, SSH, etc.) présentes

### Étape 5 – Suppression des règles legacy
- Retour dans le Migration Assistant
- Exécution de **Remove all legacy firewall rules**
- Confirmation et Apply

---

## 4. État final après migration

- Toutes les règles sont maintenant dans le **nouveau système MVC/API**
- Nombre total de règles visibles : 112 (toutes interfaces confondues)
- Règles LAN (`igc2_LAN`) propres et fonctionnelles
- Alias `STUN_TURN` et `STUN_TURN_PORTS` utilisés correctement
- Règle STUN/TURN optimisée en place
- Règle WebRTC Media Ports (50000-65535) en place
- Règle de block finale conservée
- Aucune règle critique perdue

### Points notables
- La règle temporaire « SpeedTest Ookla » a été laissée désactivée
- Les règles « Default allow LAN » restent désactivées (comportement souhaité)

---

## 5. Notes importantes

- Pas de ZFS → la seule méthode de rollback fiable reste le fichier XML de configuration + Configuration History
- Après la migration, l’édition des règles se fait uniquement dans la nouvelle interface
- Le bouton **Inspect** permet d’afficher/masquer les règles automatiquement générées

---

## 6. Conclusion

Migration réussie le 31 juillet 2026.  
Le firewall fonctionne correctement avec le nouveau moteur de règles MVC/API.  
Les accès critiques (interface web, SSH, BunkerWeb) ont été validés après la bascule.



---

