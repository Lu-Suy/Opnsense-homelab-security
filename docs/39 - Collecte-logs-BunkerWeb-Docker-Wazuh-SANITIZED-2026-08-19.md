# 39 – Collecte des logs BunkerWeb & Docker dans Wazuh

**Projet :** Bastion Godmode / Horus AIS  
**Date :** 19 août 2026  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Composants concernés :** BunkerWeb 1.6.13 + Docker + Wazuh 4.14.7 (all-in-one)  

**Objectif de cette fiche :**  
Faire remonter dans Wazuh les logs applicatifs de BunkerWeb (access + sécurité) et des conteneurs Docker associés, afin d’obtenir une visibilité réelle sur le trafic web et les événements de protection (Bad Behavior, antibot, etc.).

---

## 1. Contexte

Jusqu’à présent, Wazuh recevait principalement :
- Les logs système du bastion
- Les logs Syslog d’OPNsense
- Les événements de l’agent Windows (AlphaDeck)

Il manquait la couche applicative web.  
BunkerWeb étant déployé en Docker, ses logs passent par le driver `json-file` de Docker. L’objectif était donc de brancher ces fichiers de logs dans le Manager Wazuh.

---

## 2. Diagnostic initial

### 2.1 État des conteneurs

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

Conteneurs identifiés :
- `bunkerweb` (bunkerity/bunkerweb:1.6.13)
- `bw-scheduler` (bunkerity/bunkerweb-scheduler:1.6.13)
- `docker-socket-proxy`

### 2.2 Configuration de logging

```bash
docker inspect bunkerweb --format '{{json .HostConfig.LogConfig}}' | jq .
```

Résultat :
```json
{
  "Type": "json-file",
  "Config": {}
}
```

BunkerWeb logue donc en **stdout** → les logs sont stockés par Docker au format JSON dans :
`/var/lib/docker/containers/<ID>/<ID>-json.log`

Aucun fichier de log classique n’existe dans le volume `bunkerweb_bw-data`.

![[Pasted image 20260819054246.png]]

### 2.3 Chemin exact du fichier de log

**1. Récupérer l’ID du conteneur**
```bash
docker inspect bunkerweb --format '{{.Id}}'
```
![[Pasted image 20260819054753.png]]
**2. Voir le chemin exact du fichier de log (le plus simple)**
```bash
docker inspect -f '{{.LogPath}}' bunkerweb
```
![[Pasted image 20260819054824.png]]
**3. (optionnel mais utile) Vérifier le fichier**
```bash
ls -lh $(docker inspect -f '{{.LogPath}}' bunkerweb)
```
![[Pasted image 20260819054844.png]]
Chemin obtenu :
```text
/var/lib/docker/containers/74f7bf23824ab87b4654bff71db54dad5018e3b5a955cb5ceb017618604272bc/74f7bf23824ab87b4654bff71db54dad5018e3b5a955cb5ceb017618604272bc-json.log
```

**Point d’attention :**  
L’ID du conteneur change si le conteneur est recréé. Un chemin en dur est fragile → utilisation d’un motif avec jokers plus tard.

---

## 3. Problème de permissions

### 3.1 Constat

Le fichier de log Docker est en `root:root` avec les permissions `640`.  
Le processus Wazuh (utilisateur système `wazuh`) ne peut pas le lire.

```bash
sudo -u wazuh test -r $(docker inspect -f '{{.LogPath}}' bunkerweb) && echo "Lisible" || echo "PAS lisible"
```

Résultat : **PAS lisible**.

![[Pasted image 20260819055712.png]]
```bash
sudo -u wazuh head -n 2 $(docker inspect -f '{{.LogPath}}' bunkerweb)
```
![[Pasted image 20260819055730.png]]

### 3.2 Clarification sur l’utilisateur `wazuh`

L’utilisateur `wazuh` est un **utilisateur système** créé automatiquement par le package officiel Wazuh.  
Il ne fait pas partie des comptes administratifs créés manuellement sur le bastion (`doo`, `hyper_doo`).

### 3.3 Tentative via le groupe `docker`

```bash
usermod -aG docker wazuh
systemctl restart wazuh-manager
```

Le groupe `docker` a bien été ajouté, mais sur ce système il ne donne **pas** automatiquement l’accès en lecture aux fichiers sous `/var/lib/docker/containers/` (souvent en `700 root:root`).

![[Pasted image 20260819060713.png]]

Puis on reteste :

```bash
sudo -u wazuh ls /var/lib/docker/containers/ | head -5
sudo -u wazuh test -r $(docker inspect -f '{{.LogPath}}' bunkerweb) && echo "Maintenant lisible" || echo "Toujours pas"
```
![[Pasted image 20260819061028.png]]


### 3.4 Solution retenue : ACL

Installation du paquet si nécessaire :
```bash
dnf install -y acl
```

Application des ACL :
```bash
setfacl -m u:wazuh:rx /var/lib/docker
setfacl -m u:wazuh:rx /var/lib/docker/containers
setfacl -m u:wazuh:rx $(dirname $(docker inspect -f '{{.LogPath}}' bunkerweb))
setfacl -m u:wazuh:r  $(docker inspect -f '{{.LogPath}}' bunkerweb)
```

Vérification :
```bash
sudo -u wazuh test -r $(docker inspect -f '{{.LogPath}}' bunkerweb) && echo "Maintenant lisible" || echo "Toujours pas"
```

Résultat : **Maintenant lisible**.

![[Pasted image 20260819062547.png]]

---

## 4. Configuration Wazuh (localfile)

### 4.1 Sauvegarde et modification

```bash
cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak-$(date +%Y%m%d-%H%M)
```

Ajout du bloc suivant dans `/var/ossec/etc/ossec.conf` (avant la balise de fermeture) :

```xml
  <!-- BunkerWeb / Docker logs -->
  <localfile>
    <log_format>json</log_format>
    <location>/var/lib/docker/containers/*/*-json.log</location>
    <label key="docker">bunkerweb</label>
  </localfile>
```

**Choix techniques :**
- `log_format>json` → adapté au format Docker
- Chemin avec `*` → robuste même si le conteneur est recréé
- Label → facilite le filtrage ultérieur dans le Dashboard

### 4.2 Redémarrage et vérification

```bash
systemctl restart wazuh-manager
```

Vérifie que ça a bien pris**
```bash
tail -30 /var/ossec/logs/ossec.log | grep -i "docker\|localfile\|json\|error"
```

![[Pasted image 20260819063626.png]]

Confirmation dans les logs Wazuh :

```bash
grep -iE "Analyzing file|json.log|docker|containers" /var/ossec/logs/ossec.log | tail -20
```
![[Pasted image 20260819063828.png]]
Résultat clé :
```text
New file that matches the '/var/lib/docker/containers/*/*-json.log' pattern: '...'
Analyzing file: '/var/lib/docker/containers/74f7bf...-json.log'
```

Wazuh a bien détecté et commence à analyser les fichiers de logs Docker (y compris celui de BunkerWeb).



---

## 5. Vérification dans le Dashboard

Après ouverture du Dashboard Wazuh (`https://10.0.10.11:8443`) et recherche sur `json.log` ou `location:*json.log*`, des événements provenant des conteneurs Docker apparaissent.

Les logs sont encore bruts (format JSON Docker).  
Ils sont collectés, mais pas encore enrichis ni facilement filtrables (IP, URL, code HTTP, raison de ban, etc.).

![[Pasted image 20260819070204.png]]

---

## 6. État final et suite

| Élément                              | Statut      | Commentaire |
|--------------------------------------|-------------|-------------|
| Localisation des logs BunkerWeb      | ✅ Terminé  | Driver json-file Docker |
| Permissions (user `wazuh`)           | ✅ Terminé  | ACL appliquées |
| Configuration localfile Wazuh        | ✅ Terminé  | Motif wildcard |
| Collecte active                      | ✅ Terminé  | Confirmée dans ossec.log |
| Logs visibles dans le Dashboard      | ✅ Terminé  | Encore bruts |
| Décodeurs / règles métier            | ⏳ À faire  | Fiche 39A |

---

## 7. Checklist de clôture

- [x] Identification du chemin des logs Docker
- [x] Résolution des permissions (ACL)
- [x] Ajout du localfile dans `ossec.conf`
- [x] Redémarrage du Manager Wazuh
- [x] Confirmation que le logcollector analyse les fichiers
- [x] Vérification de présence d’événements dans le Dashboard
- [x] Documentation réalisée

---

## 8. Prochaine étape recommandée

**Fiche 39A – Décodeurs et règles pour les logs BunkerWeb**

Objectif : transformer les lignes JSON brutes en événements structurés et exploitables (extraction de l’IP réelle, de l’URL, du code HTTP, des raisons de ban Bad Behavior, etc.) afin de pouvoir créer des alertes pertinentes et des vues Dashboard utiles.

---

**Fin de la fiche 39**  
