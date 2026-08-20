# Fiche 39A – Intégration des logs BunkerWeb Bad Behavior dans Wazuh

**Date :** 20 août 2026  
**Machine :** G4 (HP EliteDesk 800 G4 – Rocky Linux 10 + LUKS)  
**Services :** BunkerWeb 1.6.13 (Docker) + Wazuh 4.14.7  
**Objectif :** Transformer les messages Bad Behavior de BunkerWeb en alertes exploitables dans le Dashboard Wazuh.

---

## 1. Contexte et objectif

Après la fiche 39 (collecte des logs Docker de BunkerWeb), les logs arrivent bien dans Wazuh mais restent **bruts**.

L’objectif de la fiche 39A est de :
- Détecter les événements `[BADBEHAVIOR]`
- Générer des alertes pertinentes
- Préparer l’extraction structurée des champs (IP réelle, action, compteur, status) pour une future fiche 39B

---

## 2. Collecte des samples de logs

### Premiers logs (trop pauvres)

```bash
docker logs --tail 100 bunkerweb 2>&1 | grep -E 'GET|POST|denied|bad.?behavior|403|antibot|BAN' | tail -20
```

Résultat : uniquement des healthchecks internes (`172.29.0.3` – bwapi). Aucun trafic externe, aucun 403, aucun Bad Behavior.

### Génération de trafic contrôlé

```bash
curl -I https://horus-ais.com/
curl -I https://horus-ais.com/wp-admin/
curl -I https://horus-ais.com/.env
curl -A "sqlmap" https://horus-ais.com/
curl -A "nikto" https://horus-ais.com/
```

### Samples utiles obtenus

- IP réelle visible : `68.66.116.176` puis `161.178.138.83` (Real IP Cloudflare OK)
- Messages `[BADBEHAVIOR] increased counter for IP ...`
- Messages ModSecurity 403
- Format d’access log BunkerWeb confirmé

![[Pasted image 20260820092408.png]]

---

## 3. Approches testées (chronologie technique)

### Approche 1 – Decoder custom indépendant

Création d’un decoder dédié :

```xml
<decoder name="bunkerweb-badbehavior">
  <prematch>BADBEHAVIOR</prematch>
  <regex>(\w+) counter for IP (\S+)</regex>
  <order>action, srcip</order>
</decoder>
```

**Résultat :** Échec  
Le decoder natif `nginx-errorlog` restait prioritaire.  
`wazuh-logtest` montrait systématiquement `name: 'nginx-errorlog'`.

![[Pasted image 20260820103205.png]]

### Approche 2 – Méthode officielle Wazuh (child sous nginx-errorlog)

1. Copie du decoder nginx natif :
```bash
# 1. On copie le decoder nginx officiel dans notre répertoire custom
sudo cp /var/ossec/ruleset/decoders/0170-nginx_decoders.xml /var/ossec/etc/decoders/0170-nginx_decoders.xml

# 2. On met les bons droits (propriétaire wazuh + permissions correctes)
sudo chown wazuh:wazuh /var/ossec/etc/decoders/0170-nginx_decoders.xml
sudo chmod 660 /var/ossec/etc/decoders/0170-nginx_decoders.xml
```
### Vérification

```bash
ls -l /var/ossec/etc/decoders/0170-nginx_decoders.xml
```
``
![[Pasted image 20260820110414.png]]

2. Exclusion de l’original dans `ossec.conf` :
```xml
<decoder_exclude>ruleset/decoders/0170-nginx_decoders.xml</decoder_exclude>
```

3. Ajout d’un child decoder sous `nginx-errorlog`.

**Résultat :** Échec  
Le moteur de regex de Wazuh 4.14.7 est extrêmement strict.  
Toute utilisation de `\[` ou de regex un peu complexes provoquait :
```
ERROR: (2107): Decoder configuration error
```

De plus, l’exclusion + suppression de la copie a cassé les règles nginx natives (`Invalid decoder name: 'nginx-errorlog'`).

![[Pasted image 20260820105707.png]]



### Approche 3 – Solution pragmatique (règle) — **REtenue**

On laisse le decoder natif `nginx-errorlog` faire son travail et on crée une **règle** qui se déclenche sur la présence de `BADBEHAVIOR`.

**Résultat :** Succès

---

## 4. Solution retenue

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Ajoute **à la fin** du fichier (avant la dernière balise de fermeture s’il y en a une) :

```xml
<!-- ===================================================== -->
<!-- BunkerWeb Bad Behavior - Fiche 39A                    -->
<!-- ===================================================== -->
<group name="bunkerweb,">
  <rule id="100200" level="7">
    <if_sid>31300</if_sid>
    <match>BADBEHAVIOR</match>
    <description>BunkerWeb Bad Behavior: possible attack detected</description>
    <group>web,attack,</group>
  </rule>
</group>
```


```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```
### Test de validation (`wazuh-logtest`)

On va vérifier que les champs sortent correctement.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Une fois dans l’outil, colle **cette ligne réelle** (tirée de tes logs) :

```text
2026/08/20 13:43:00 [notice] 449#449: *116044 [BADBEHAVIOR] increased counter for IP 161.178.138.83 (5/10) on server horus-ais.com (status 403, scope service), context: ngx.timer
```

```text
**Phase 2: Completed decoding.
        name: 'nginx-errorlog'
**Phase 3: Completed filtering (rules).
        id: '100200'
        level: '7'
        description: 'BunkerWeb Bad Behavior: possible attack detected'
        groups: '['bunkerweb', 'web', 'attack']'
**Alert to be generated.
```

![[Pasted image 20260820113140.png]]

---

## 5. État actuel

- Alerte niveau 7 générée dès qu’un message Bad Behavior apparaît
- Solution stable et non destructive
- Aucun risque de casser les decoders/règles natives

### Limitations actuelles
- Pas encore d’extraction structurée des champs (IP, action, compteur, status)
- L’alerte reste générique

---

## 6. Récapitulatif des leçons

| Approche | Idée | Résultat |
|---------|------|----------|
| 1. Decoder indépendant | Créer notre propre decoder | Échec → nginx gagnait toujours |
| 2. Méthode officielle (child) | Modifier le decoder nginx | Échec → moteur de regex trop strict |
| 3. Règle (actuelle) | Laisser nginx + ajouter une règle | **Succès** |

---

## 7. Prochaines étapes (Fiche 39B)

1. Enrichir la règle pour différencier `increased` / `decreased`
2. Extraire l’IP source dans la description de l’alerte
3. Créer des règles de niveau plus élevé quand le compteur approche du seuil
4. Ajouter des règles pour les blocages ModSecurity
5. Documenter les vues Dashboard utiles

---

**Fin de la fiche 39A – 20 août 2026**