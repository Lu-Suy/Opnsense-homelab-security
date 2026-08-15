# 30a – Paramétrage initial Wazuh (G4)

**Projet :** Horus AIS – bastion  
**Date :** 12 août 2026 (portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Stack :** Wazuh 4.14.7 (Indexer + Manager + Dashboard + Filebeat)  
**Dashboard :** `https://10.0.10.y:8443`  

**Prérequis :** fiche **30 – Installation Wazuh SIEM (G4)** terminée et validée.

**Objectif :** sécuriser les comptes par défaut, valider la chaîne Filebeat → Indexer, documenter la tentative d’agent local (échec attendu en all-in-one), et figer les décisions d’architecture pour la suite.

> **Publication :** IP `10.0.10.y`. **Aucun mot de passe réel** — uniquement des placeholders. Les secrets vivent dans le coffre personnel / le keystore.

---

## 1. Changement du mot de passe admin

### 1.1 Piège UI classique

Depuis le Dashboard (*Reset password*), tentative de changement du compte `admin` :

```text
Failed to reset password.
{"status":"FORBIDDEN","message":"Resource 'admin' is reserved."}
```

![Erreur admin reserved](../images/Pasted%20image%2020260812090733.png)

**Explication :**  
`admin` n’est **pas** un utilisateur métier. C’est un compte **réservé** du plugin OpenSearch Security.  
Il ne se change **pas** via l’interface web — ce n’est pas un bug de la config lab.

### 1.2 Méthode officielle (outil Wazuh)

```bash
# Génère / applique des mots de passe pour tous les comptes techniques OpenSearch Security
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -a
```

(selon packaging : éventuellement préfixer par `bash`.)

L’option `-a` touche **tous** les comptes techniques et les affiche une fois.  
**Les noter immédiatement dans un coffre** — pas seulement dans le terminal.

| Utilisateur | Rôle |
|-------------|------|
| **admin** | Admin Dashboard / Indexer (login UI) |
| **kibanaserver** | Dashboard → Indexer |
| **kibanaro** | Lecture seule Dashboard |
| **logstash** / côté Filebeat | Ingestion |
| **readall** | Lecture seule technique |
| **snapshotrestore** | Snapshots |
| **anomalyadmin** | Anomaly detection |

Ce ne sont **pas** des comptes humains “métier”.

### 1.3 Mot de passe choisi (option lab)

Pour un secret **maîtrisé** (coffre) plutôt que purement aléatoire :

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin \
  -p 'MOT_DE_PASSE_FORT_COFFRE'
```

**Contraintes usuelles :** ≥ 8 caractères, majuscule, minuscule, chiffre, caractère spécial. **Pas d’espace.**

| Approche | Quand |
|----------|--------|
| `-a` (aléatoire fort) | Très bien lab + pro |
| `-u admin -p '…'` | Si le secret doit être dans ton coffre personnel |

Dans les deux cas : **ne plus laisser `admin` / `admin`**.

---

## 2. Synchronisation Filebeat (keystore)

Le keystore Filebeat stocke les identifiants de façon **chiffrée** (pas en clair dans `filebeat.yml`).

```bash
echo 'admin' | sudo filebeat keystore add username --stdin --force
echo 'MOT_DE_PASSE_FORT_COFFRE' | sudo filebeat keystore add password --stdin --force

sudo systemctl restart filebeat
sudo filebeat test output
```

Résultat attendu :

```text
TLS handshake... OK
talk to server... OK
```

![filebeat test output OK](../images/Pasted%20image%2020260812091916.png)

### Problème rencontré – `401 Unauthorized`

**Cause typique :** le mot de passe dans le **keystore** ne correspond **pas** (encore) à celui appliqué côté Indexer (décalage après `-a` puis `-u admin`, typo, ancien secret).

**Remède :** réécrire **exactement** le même couple `admin` / secret dans le keystore, puis `restart` + `filebeat test output`.

### Stockage du secret

| Endroit | Statut |
|---------|--------|
| Keystore Filebeat | **Chiffré** |
| `filebeat.yml` | Références au keystore, pas le mdp en clair |
| Historique shell (`~/.zsh_history`) | Peut contenir le secret si tapé en clair → nettoyer si besoin |

---

## 3. Validation accès Dashboard

| Élément | Statut |
|---------|--------|
| URL | `https://10.0.10.y:8443` |
| User | `admin` |
| Password | secret coffre (aligné outil + keystore) |
| Login | **OK** |
| Overview | Visible (agents = 0, alertes système présentes) |

![Login Dashboard OK](../images/Pasted%20image%2020260812092358.png)

**Note :** le compte API Manager (`wazuh-wui`) n’est **pas** modifié par l’outil si les credentials API ne sont pas fournis (`Wazuh API admin credentials not provided`). **Non bloquant** pour l’UI ; à traiter plus tard si besoin.

Des alertes *medium* / *low* apparaissent déjà **sans agent** : bruit et règles par défaut du Manager — normal.

---

## 4. Qu’est-ce qu’un agent Wazuh ? (rappel)

L’**agent** est le programme installé **sur la machine à surveiller**. Il envoie au Manager notamment :

- logs système ;
- intégrité des fichiers (FIM) ;
- process / ports ;
- modules selon config (rootkit, etc.).

Sans agent (ou sans syslog / `localfile`), le Manager ne voit quasi que **lui-même**.  
Avec des agents (ou syslog), on passe à une vraie collecte multi-sources.

### Plan de déploiement (ordre lab)

| Ordre | Source | Mode | Statut |
|-------|--------|------|--------|
| — | Host Manager (G4) | Paquet `wazuh-agent` | **Non** (conflit — § 5) |
| 1 | OPNsense | **Syslog** → Manager | Prochaine fiche |
| 2 | AlphaDeck (Windows) | Agent Windows | Plus tard |
| 3 | BunkerWeb / Docker | Logs locaux / `localfile` | À configurer |
| 4 | Honeypot / autres | Agent ou syslog | Plus tard |

---

## 5. Tentative d’agent sur le G4 — échec documenté

**Réflexe pro :** “le bastion se surveille lui-même”. Légitime — mais **incompatible** avec le packaging RPM all-in-one actuel.

### 5.1 Commande générée par l’UI

```bash
curl -o wazuh-agent-4.14.7-1.x86_64.rpm \
  https://packages.wazuh.com/4.x/yum/wazuh-agent-4.14.7-1.x86_64.rpm \
  && sudo WAZUH_MANAGER='10.0.10.y' WAZUH_AGENT_NAME='Agent_G4' \
     rpm -ihv wazuh-agent-4.14.7-1.x86_64.rpm
```

![Deploy agent UI](../images/Pasted%20image%2020260812094401.png)

### 5.2 Résultat

```text
error: Failed dependencies:
  wazuh-manager conflicts with wazuh-agent-4.14.7-1.x86_64
  wazuh-agent conflicts with (installed) wazuh-manager-4.14.7-1.x86_64
```

### 5.3 Pourquoi ça n’aboutit pas

Sur une install **all-in-one** (Indexer + Manager + Dashboard sur la même machine), les paquets RPM `wazuh-manager` et `wazuh-agent` **s’excluent mutuellement**. Comportement voulu par Wazuh.

| Approche | Verdict |
|----------|---------|
| Forcer avec `--nodeps` / `--force` | **Non** — sale, non pro, risque de casse |
| Agent sur le host Manager | **Non supporté** proprement en RPM all-in-one |
| Surveillance du G4 | Partielle via logs du Manager + futurs `localfile` / modules host |

Nettoyage :

```bash
rm -f wazuh-agent-4.14.7-1.x86_64.rpm
```

---

## 6. Décisions d’architecture (figées)

| Sujet | Décision |
|-------|----------|
| Agent sur G4 (host Manager) | **Non** — conflit de paquets |
| Premier agent externe | AlphaDeck (Windows) — plus tard |
| Monitoring OPNsense | **Syslog** → Manager (pas d’agent FreeBSD simple) |
| Monitoring BunkerWeb | Logs Docker / fichiers (même host) |
| Box FAI | **Pas d’agent** — équipement fermé |
| All-in-one sur G4 | **Conservé** pour le lab actuel (16 Go, charge acceptable) |
| SIEM dédié | Évolution possible si charge / isolation le justifient |

### Schéma de collecte cible

```text
OPNsense ──syslog──► Wazuh Manager (G4)
AlphaDeck ──agent──► Wazuh Manager (G4)
BunkerWeb (Docker) ─logs locaux─► Manager
G4 (host) ─logs Manager + localfile─► (sans paquet agent)
```

---

## 7. État de fin de paramétrage initial

| Composant | Statut |
|-----------|--------|
| Indexer | UP, security GREEN |
| Manager | UP |
| Filebeat | UP, `test output` OK |
| Dashboard | UP sur **8443**, login OK |
| Mot de passe admin | Changé (outil + keystore alignés) |
| Agents enregistrés | **0** (attendu) |
| Agent G4 | Tentative échouée / documentée |
| firewalld | 8443, 1514, 1515, 55000 |
| OPNsense LAN | Règles Dashboard / API depuis poste admin |

---

## 8. Suite recommandée

1. **Syslog OPNsense → Wazuh** (meilleur ROI visibilité firewall) — fiche **31**  
2. Agent **Windows** sur AlphaDeck — fiche **32**  
3. Collecte logs **BunkerWeb / Docker**  
4. Renforcement `localfile` / FIM côté Manager pour le G4  
5. (Optionnel) Changement mot de passe API Manager  

---

## 9. Checklist 30a

- [x] Comprendre que `admin` est réservé (pas de reset UI)
- [x] Changer le mdp admin via `wazuh-passwords-tool.sh`
- [x] Aligner le keystore Filebeat
- [x] `filebeat test output` → OK
- [x] Login Dashboard avec nouveau mdp
- [x] Tentative agent G4 → conflit RPM documenté
- [x] Décision : pas d’agent sur host Manager
- [ ] Syslog OPNsense (fiche 31)
- [ ] Agent AlphaDeck (fiche 32)

---

## Références

- Fiche **30** – Installation Wazuh SIEM (G4)  
- Fiche **04b** – Firewall Rules Hardening  
- Fiches **28** / **29** – SSH + Fail2ban  
- Documentation Wazuh 4.x – password tool / all-in-one  

---

**Document portfolio / runbook – 15 août 2026**
