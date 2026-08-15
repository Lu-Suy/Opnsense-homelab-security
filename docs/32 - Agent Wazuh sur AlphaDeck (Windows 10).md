# 32 – Agent Wazuh sur AlphaDeck (Windows 10)

**Projet :** Horus AIS – bastion  
**Date :** 12 août 2026 (fusion portfolio 15 août 2026)  
**Machine cible :** AlphaDeck (Windows 10) – poste d’administration  
**Manager :** G4 / hostname `bastion` – `10.0.10.y` – Wazuh **4.14.7**  
**Méthode :** installation manuelle (MSI)

**Objectif :** enroller un agent Windows pour remonter les événements de sécurité du poste admin et compléter le SOC local (OPNsense + G4 + AlphaDeck).

> **Publication :** IP management `10.0.10.y`, LAN admin `10.0.0.x`. Hostname `bastion`.

---

## 1. Prérequis

| Élément | Valeur / état |
|---------|----------------|
| OS | Windows 10 |
| IP AlphaDeck | `10.0.0.x` (lab) |
| Manager Wazuh | `10.0.10.y` (G4) |
| Version agent | **4.14.7** (identique au manager) |
| Droits | Administrateur sur AlphaDeck |
| Réseau | LAN → OPT1 autorisé (ports agents déjà ouverts côté fiche 30 : 1514 / 1515) |

---

## 2. Téléchargement

Sur AlphaDeck, MSI officiel :

```text
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi
```

Toujours aligner la version **majeure/mineure** avec le manager.

---

## 3. Installation

1. Lancer `wazuh-agent-4.14.7-1.msi`  
2. Suivre l’assistant  
3. En fin d’install, **cocher** « Run Agent configuration interface »  
4. **Finish**

![Installation MSI / interface agent](../images/Pasted%20image%2020260812153912.png)

---

## 4. Configuration (enrollment)

Dans l’interface de configuration :

- **Manager IP** → `10.0.10.y`  
- **Authentication key** → générée sur le manager  

### 4.1 Génération de la clé (G4)

```bash
sudo /var/ossec/bin/manage_agents
```

Menu interactif :

1. `A` → Add agent  
2. Nom : `Agent_AlphaDeck`  
3. IP : IP LAN d’AlphaDeck (ou `any`)  
4. `E` → Extract key pour cet agent  
5. Copier **toute** la clé affichée  

Coller la clé dans **Authentication key** côté Windows → **Save**.

![Enrollment clé manager](../images/Pasted%20image%2020260812154244.png)

> La fenêtre peut ne pas se fermer après Save — **normal**.

---

## 5. Démarrage du service

PowerShell ou Invite de commandes **en Administrateur** :

```cmd
NET START WazuhSvc
```

Attendu :

```text
The Wazuh service was started successfully.
```

![Service WazuhSvc démarré](../images/Pasted%20image%2020260812155249.png)

Le service tourne en arrière-plan : on peut fermer l’interface de configuration.

Alternative : Services Windows → service **Wazuh** → Démarrer.

---

## 6. Vérification côté Manager

```bash
sudo /var/ossec/bin/agent_control -l
```

Exemple de résultat lab :

```text
ID: 000, Name: bastion (server), IP: 127.0.0.1, Active/Local
ID: 001, Name: Agent_AlphaDeck, IP: 10.0.0.x, Active
```

Aussi visible dans le Dashboard → **Agents**.

![Agent Active dans le Dashboard](../images/Pasted%20image%2020260812155454.png)

Détail agent :

```bash
sudo /var/ossec/bin/agent_control -i 001
```

---

## 7. Premiers événements & filtres Discover

### 7.1 Filtre de base

```text
agent.name: Agent_AlphaDeck
```

### 7.2 Exclure le bruit SCA (recommandé)

Le premier scan SCA (CIS) génère beaucoup d’événements. Pour ne garder que les événements “métier” :

```text
agent.name: Agent_AlphaDeck AND NOT decoder.name: sca
```

Lab : ~6 événements propres (démarrage agent, connexion manager, Windows Logon Success…).

![Discover hors SCA](../images/Pasted%20image%2020260812160622.png)

### 7.3 Autres filtres utiles

| Objectif | Filtre |
|----------|--------|
| Connexions réussies | `agent.name: Agent_AlphaDeck AND rule.id: 60106` |
| Canal Security | `agent.name: Agent_AlphaDeck AND data.win.system.channel: Security` |
| Tout sauf SCA (groupe) | `agent.name: Agent_AlphaDeck AND NOT rule.groups: sca` |

**Astuce :** **Save** (Saved Search), pas seulement le pin.

Suivi live côté G4 (optionnel) :

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep -i alphadeck
```

---

## 8. Vie privée / périmètre de collecte

Par défaut, l’agent Windows :

- remonte les **logs de sécurité** Windows ;  
- lance un **scan SCA** (configuration) ;  
- surveille certains chemins système (FIM) ;  
- **ne lit pas** automatiquement documents personnels, photos, historique navigateur, etc.

On peut restreindre plus tard (désactiver FIM large, limiter les canaux d’événements, etc.).

---

## 9. Checklist

- [x] MSI 4.14.7 installé  
- [x] Manager IP configuré  
- [x] Clé d’authentification générée et importée  
- [x] Service `WazuhSvc` démarré  
- [x] Agent **Active** (ID 001)  
- [x] Premiers événements dans Discover  
- [x] Filtre hors SCA testé  

---

## 10. Suite possible

- Affiner l’agent (volume SCA, modules)  
- Saved Searches / vues dédiées AlphaDeck  
- Simulations contrôlées (fiche **31A**) avec poste déjà monitoré  
- Collecte logs BunkerWeb / Docker sur le G4  

---

## Références

- Fiche **30** / **30a** – Wazuh all-in-one + paramétrage  
- Fiche **31** – Syslog OPNsense  
- Ports agents (1514 / 1515) ouverts lors de l’install Wazuh  

---

**Document fusionné – version portfolio / runbook – 15 août 2026**  
**Statut lab :** agent AlphaDeck opérationnel (12 août 2026)
