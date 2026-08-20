# Fiche 39B – Decoder PCRE2 Bad Behavior + Règle enrichie (Wazuh)

**Date :** 20 août 2026  
**Machine :** G4 (HP EliteDesk 800 G4 – Rocky Linux 10 + LUKS)  
**Services :** BunkerWeb 1.6.13 (Docker) + Wazuh 4.14.7  
**Objectif :** Extraire de manière structurée les champs des logs Bad Behavior de BunkerWeb et enrichir les alertes Wazuh.

---

## 1. Contexte et lien avec la fiche 39A

Après plusieurs recherches et tentatives documentées dans la fiche 39A, une résolution propre au problème d’extraction des champs Bad Behavior a été trouvée.

Dans la fiche 39A, trois approches avaient été testées :

1. **Decoder indépendant**  
   Création d’un decoder sans parent.  
   **Résultat :** Échec. Le decoder natif `nginx-errorlog` restait prioritaire.  
   `wazuh-logtest` affichait systématiquement `name: 'nginx-errorlog'`.

2. **Méthode officielle Wazuh (child sous nginx-errorlog)**  
   - Copie du fichier natif `0170-nginx_decoders.xml`  
   - Exclusion de l’original via `<decoder_exclude>`  
   - Ajout d’un child decoder  
   **Résultat :** Échec.  
   Le moteur de regex de Wazuh 4.14.7 est extrêmement strict. Toute utilisation de `\[` ou de regex un peu complexes provoquait l’erreur :
   ```
   ERROR: (2107): Decoder configuration error
   ```
   De plus, l’exclusion + suppression de la copie a cassé les règles nginx natives (`Invalid decoder name: 'nginx-errorlog'`).

3. **Solution pragmatique (règle seule)**  
   Laisser le decoder natif `nginx-errorlog` et créer une règle basée sur `<if_sid>31300</if_sid>` + `<match>BADBEHAVIOR</match>`.  
   **Résultat :** Succès.  
   Une alerte de niveau 7 était générée, mais sans extraction structurée des champs (IP, action, compteur, status).

L’objectif de la fiche 39B est de passer de cette détection générique à une **extraction structurée** des champs tout en restant non destructif.

---

## 2. Point clé technique

Le secret résidait dans l’utilisation de **`type="pcre2"`**.

Le moteur classique de Wazuh (OS_Regex) refusait les crochets `\[` dans les expressions régulières.  
Avec le moteur PCRE2, le parsing fonctionne correctement.

C’est ce changement qui a permis de créer un decoder child fonctionnel sous `nginx-errorlog` sans modifier ni exclure les decoders natifs.

---

## 3. Principes de sécurité appliqués

Afin de ne rien casser de l’existant, les principes suivants ont été strictement respectés :

- Ne **pas** toucher au fichier natif `0170-nginx_decoders.xml`
- Ne **pas** utiliser de balise `<decoder_exclude>`
- Créer **uniquement** un nouveau fichier custom
- Tester systématiquement avec `wazuh-logtest` **avant** de redémarrer le manager
- Conserver la règle 100200 existante comme filet de sécurité pendant toute la phase de test

Ces précautions garantissent que même en cas d’échec du nouveau decoder, la détection de base reste opérationnelle.

---

## 4. Mise en place du decoder custom

### 4.1 Création du fichier

```bash
sudo nano /var/ossec/etc/decoders/0390-bunkerweb_decoders.xml
```

Contenu exact placé dans le fichier :

```xml
<!-- ===================================================== -->
<!-- BunkerWeb Bad Behavior - Custom Decoder (Fiche 39B)   -->
<!-- ===================================================== -->

<decoder name="bunkerweb-badbehavior">
  <parent>nginx-errorlog</parent>
  <prematch type="pcre2">\[BADBEHAVIOR\]</prematch>
  <regex type="pcre2">\[BADBEHAVIOR\] (\w+) counter for IP (\S+) \((\d+)/(\d+)\).*status (\d+)</regex>
  <order>action, srcip, counter, threshold, status</order>
</decoder>
```

### 4.2 Attribution des droits

```bash
sudo chown wazuh:wazuh /var/ossec/etc/decoders/0390-bunkerweb_decoders.xml
sudo chmod 660 /var/ossec/etc/decoders/0390-bunkerweb_decoders.xml
```

### 4.3 Nettoyage de l’ancien fichier incomplet

Un ancien fichier `50-bunkerweb_decoders.xml` (version incomplète sans parent et sans extraction complète des champs) était présent. Il a été supprimé pour éviter tout conflit :

```bash
sudo rm /var/ossec/etc/decoders/50-bunkerweb_decoders.xml
```

État final du répertoire après nettoyage :

```bash
sudo ls -la /var/ossec/etc/decoders/
```

Résultat :
- `0390-bunkerweb_decoders.xml` (version complète et fonctionnelle)
- `local_decoder.xml` (fichier par défaut)

---

## 5. Validation du decoder (wazuh-logtest)

Commande de test :

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Ligne de log utilisée (issue des samples générés lors de la fiche 39A) :

```text
2026/08/20 13:43:00 [notice] 449#449: *116044 [BADBEHAVIOR] increased counter for IP 161.178.138.83 (5/10) on server horus-ais.com (status 403, scope service), context: ngx.timer
```

**Résultat obtenu :**

```text
**Phase 2: Completed decoding.
        name: 'nginx-errorlog'
        parent: 'nginx-errorlog'
        action: 'increased'
        counter: '5'
        srcip: '161.178.138.83'
        status: '403'
        threshold: '10'
```

![[Pasted image 20260820123359.png]]

Les cinq champs demandés (`action`, `srcip`, `counter`, `threshold`, `status`) sont correctement extraits.  
Même si le nom affiché reste celui du parent `nginx-errorlog`, les champs dynamiques provenant du child decoder sont présents et exploitables. C’est le comportement attendu et le succès technique recherché.

---

## 6. Enrichissement de la règle 100200

Modification de `/var/ossec/etc/rules/local_rules.xml` :

La règle a été remplacée par la version suivante afin d’utiliser les champs extraits :

```xml
<!-- ===================================================== -->
<!-- BunkerWeb Bad Behavior - Fiche 39B                    -->
<!-- ===================================================== -->
<group name="bunkerweb,">
  <rule id="100200" level="7">
    <if_sid>31300</if_sid>
    <match>BADBEHAVIOR</match>
    <description>BunkerWeb Bad Behavior: $(action) counter for IP $(srcip) ($(counter)/$(threshold)) - HTTP $(status)</description>
    <group>web,attack,</group>
  </rule>
</group>
```

### Validation après enrichissement de la règle

Nouveau test avec `wazuh-logtest` (même ligne de log) :

```text
**Phase 3: Completed filtering (rules).
        id: '100200'
        level: '7'
        description: 'BunkerWeb Bad Behavior: increased counter for IP 161.178.138.83 (5/10) - HTTP 403'
        groups: '['bunkerweb', 'web', 'attack']'
```

![[Pasted image 20260820124625.png]]

La description de l’alerte contient désormais toutes les informations structurées : action, adresse IP source, compteur, seuil et code HTTP.

---

## 7. Application définitive

Une fois les tests validés :

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

![[Pasted image 20260820124846.png]]

Le service est repassé correctement en état `active (running)`.

---

## 8. État final

| Élément                                      | Statut          |
|----------------------------------------------|-----------------|
| Decoder custom `0390-bunkerweb_decoders.xml` | Opérationnel    |
| Extraction des 5 champs                      | Opérationnelle  |
| Règle 100200 enrichie                        | Opérationnelle  |
| Ancien decoder `50-bunkerweb_decoders.xml`   | Supprimé        |
| Manager redémarré                            | OK              |
| Détection de base (fiche 39A) conservée      | Oui             |

---

## 9. Prochaines améliorations possibles

- Différencier les alertes `increased` et `decreased`
- Augmenter le niveau d’alerte lorsque le compteur approche du seuil (ex. 8/10 ou 9/10)
- Ajouter des règles spécifiques pour les événements ModSecurity
- Créer des visualisations Dashboard basées sur les nouveaux champs

---

**Fin de la fiche 39B – 20 août 2026**
