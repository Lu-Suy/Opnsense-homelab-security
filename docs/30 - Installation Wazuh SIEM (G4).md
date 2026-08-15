# 30 – Installation Wazuh SIEM (G4)

**Projet :** Horus AIS – bastion  
**Date :** 11–12 août 2026 (fusion portfolio 15 août 2026)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion`  
**Version Wazuh :** 4.14.7  
**Réseau management :** `10.0.10.y/24`  

**Objectif :** déployer une stack Wazuh **single-node** (Indexer + Manager + Filebeat + Dashboard) sur le bastion G4, accessible depuis le poste d’administration via OPNsense — **sans** exposer le Dashboard sur Internet.

**Hors scope de cette fiche :** changement du mot de passe admin, agents distants, règles métier → voir **30a – Paramétrage initial Wazuh**.

> **Publication :** IP management masquée en `10.0.10.y`. Hostname `bastion`.

---

## Architecture retenue

| Composant | Rôle | Port principal |
|-----------|------|----------------|
| **Wazuh Indexer** | Stockage / indexation (OpenSearch) | `9200` (local) |
| **Wazuh Manager** | Analyse, règles, alertes | `1514`, `1515`, `55000` |
| **Filebeat** | Ponte Manager → Indexer | — |
| **Wazuh Dashboard** | Console web SOC | **`8443`** |

**Ordre d’installation obligatoire :** Indexer → certificats → Manager → Filebeat → Dashboard.

**Décision port Dashboard :** le port **443** est déjà pris par **BunkerWeb** → Dashboard sur **`8443`**.

---

## 0. Préparation système

### 0.1 Vérifications

```bash
whoami
free -h
df -h /
hostname -I
sysctl vm.max_map_count
```

### 0.2 `vm.max_map_count` (exigence OpenSearch / Indexer)

L’Indexer est sensible à cette limite. Minimum courant documenté : **262144**.

**Problème lab (typo) :**  
`vm._map_count` (underscore mal placé) → commande invalide.  
La bonne clé est `vm.max_map_count`.

**Constat lab :** la valeur était déjà **1048576** (largement au-dessus du minimum) → **rien à forcer**.

Optionnel (rendre explicite et persistant) :

```bash
echo "vm.max_map_count=1048576" | sudo tee /etc/sysctl.d/99-wazuh.conf
```

### 0.3 Dépendances

```bash
sudo dnf install -y coreutils curl tar gnupg2
```

---

## 1. Dépôt officiel Wazuh

```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

sudo tee /etc/yum.repos.d/wazuh.repo > /dev/null << 'EOF'
[wazuh]
name=EL-$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
priority=1
EOF

sudo dnf clean all
sudo dnf makecache
```

![Dépôt Wazuh pris en compte](../images/Pasted%20image%2020260811094106.png)

---

## 2. Installation de l’Indexer

```bash
sudo dnf install -y wazuh-indexer
rpm -q wazuh-indexer
ls -la /etc/wazuh-indexer/
```

L’**Indexer** est le premier composant : sans lui, le Manager n’a pas de stockage interrogeable pour les alertes.

![Indexer installé](../images/Pasted%20image%2020260811094248.png)

---

## 3. Génération des certificats TLS

Wazuh exige du TLS entre composants. Génération **single-node** :

```bash
sudo curl -sO https://packages.wazuh.com/4.x/wazuh-certs-tool.sh
sudo curl -sO https://packages.wazuh.com/4.x/config.yml
cat config.yml
```

![config.yml certificats](../images/Pasted%20image%2020260811095112.png)

```bash
sudo bash ./wazuh-certs-tool.sh -A
sudo ls -la wazuh-certificates/
```

`-A` génère l’ensemble (root CA, admin, nœud indexer, filebeat/server, dashboard).

![Liste des certificats générés](../images/Pasted%20image%2020260811095518.png)

---

## 4. Déploiement des certificats Indexer

```bash
sudo mkdir -p /etc/wazuh-indexer/certs

sudo cp wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/
sudo cp wazuh-certificates/node-1.pem /etc/wazuh-indexer/certs/
sudo cp wazuh-certificates/node-1-key.pem /etc/wazuh-indexer/certs/
sudo cp wazuh-certificates/admin.pem /etc/wazuh-indexer/certs/
sudo cp wazuh-certificates/admin-key.pem /etc/wazuh-indexer/certs/
```

### Problème rencontré – noms de fichiers vs config par défaut

La config Indexer attend souvent `indexer.pem` / `indexer-key.pem`, alors que l’outil produit `node-1.pem` / `node-1-key.pem`.

**Correction :**

```bash
sudo mv /etc/wazuh-indexer/certs/node-1.pem /etc/wazuh-indexer/certs/indexer.pem
sudo mv /etc/wazuh-indexer/certs/node-1-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
```

### Droits

Sous **Zsh**, un `chmod …/*` peut mal se comporter selon le contexte. Méthode fiable :

```bash
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
sudo chmod 500 /etc/wazuh-indexer/certs
sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'
sudo ls -la /etc/wazuh-indexer/certs/
```

![Certificats Indexer en place](../images/Pasted%20image%2020260811100624.png)

---

## 5. Configuration Indexer + démarrage

```bash
# Optionnel : binder explicitement sur l’IP management
sudo sed -i 's/network.host: "0.0.0.0"/network.host: "10.0.10.y"/' /etc/wazuh-indexer/opensearch.yml

sudo grep -E 'network.host|pemcert|pemkey|nodes_dn' /etc/wazuh-indexer/opensearch.yml
```

![Config opensearch.yml](../images/Pasted%20image%2020260811101652.png)

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-indexer
sudo systemctl status wazuh-indexer --no-pager
```

Le premier démarrage peut prendre **1–2 minutes** (JVM + init). RAM Indexer observée en lab : ~1,3 Go — normal.

![Indexer active](../images/Pasted%20image%2020260811101942.png)

### Initialisation sécurité

```bash
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

Résultat attendu :

```text
Clusterstate: GREEN
Number of nodes: 1
Done with success
```

Sans cette étape, l’Indexer tourne mais le modèle sécurité (users / rôles) n’est pas correctement initialisé pour les autres composants.

![Security init GREEN](../images/Pasted%20image%2020260811102151.png)

---

## 6. Installation du Manager

```bash
sudo dnf install -y wazuh-manager
rpm -q wazuh-manager
```

Le Manager reçoit les événements, applique les règles, génère les alertes et s’appuie sur Filebeat pour les pousser vers l’Indexer.

![Manager installé](../images/Pasted%20image%2020260811102541.png)

---

## 7. Filebeat (ponte Manager → Indexer)

### Certificats Filebeat

```bash
sudo mkdir -p /etc/filebeat/certs

sudo cp wazuh-certificates/root-ca.pem /etc/filebeat/certs/
sudo cp wazuh-certificates/wazuh-1.pem /etc/filebeat/certs/filebeat.pem
sudo cp wazuh-certificates/wazuh-1-key.pem /etc/filebeat/certs/filebeat-key.pem

sudo bash -c 'chmod 500 /etc/filebeat/certs; chmod 400 /etc/filebeat/certs/*'
sudo chown -R root:root /etc/filebeat/certs
sudo ls -la /etc/filebeat/certs/
```

### Installation + config officielle 4.14

```bash
sudo dnf install -y filebeat

# Template adapté à la branche 4.14 (éviter une URL générique incorrecte)
sudo curl -so /etc/filebeat/filebeat.yml \
  https://packages.wazuh.com/4.14/tpl/wazuh/filebeat/filebeat.yml

# Pointer vers l’Indexer local (management)
sudo sed -i 's/127.0.0.1/10.0.10.y/' /etc/filebeat/filebeat.yml

# Module Wazuh
sudo curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.4.tar.gz \
  | sudo tar -xvz -C /usr/share/filebeat/module
```

### Keystore + test

```bash
sudo filebeat keystore create
echo admin | sudo filebeat keystore add username --stdin --force
echo admin | sudo filebeat keystore add password --stdin --force

sudo systemctl enable --now filebeat
sudo filebeat test output
```

Résultat attendu :

```text
TLS handshake... OK
talk to server... OK
```

![Filebeat test output OK](../images/Pasted%20image%2020260811143844.png)

---

## 8. Démarrage du Manager

```bash
sudo systemctl enable --now wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

![Manager active](../images/Pasted%20image%2020260811144256.png)

**Note RAM lab :** Manager + Indexer pèsent déjà plusieurs Go ; sur 16 Go dual-channel c’est tenable en single-node lab/pro léger, à surveiller (pic au boot).

---

## 9. Installation du Dashboard

```bash
sudo dnf install -y wazuh-dashboard
rpm -q wazuh-dashboard
```

![Dashboard installé](../images/Pasted%20image%2020260811144633.png)

### Certificats Dashboard

```bash
sudo mkdir -p /etc/wazuh-dashboard/certs

sudo cp wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/
sudo cp wazuh-certificates/dashboard.pem /etc/wazuh-dashboard/certs/
sudo cp wazuh-certificates/dashboard-key.pem /etc/wazuh-dashboard/certs/

sudo bash -c 'chmod 500 /etc/wazuh-dashboard/certs; chmod 400 /etc/wazuh-dashboard/certs/*'
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
sudo ls -la /etc/wazuh-dashboard/certs/
```

### Config : Indexer + **port 8443**

```bash
sudo sed -i 's|https://localhost:9200|https://10.0.10.y:9200|' \
  /etc/wazuh-dashboard/opensearch_dashboards.yml
```

### Problème rencontré – port 443 déjà pris

**Symptôme :** service `active` mais *Scheduled restart job, restart counter is at N* ; logs :

```text
Error: Port 443 is already in use
```

**Cause :** **BunkerWeb** (et/ou Docker) occupe déjà **443**.  
Le Dashboard Wazuh ne doit **pas** reprendre ce port.

**Correction :**

```bash
sudo systemctl stop wazuh-dashboard
sudo sed -i 's/server.port: 443/server.port: 8443/' \
  /etc/wazuh-dashboard/opensearch_dashboards.yml
sudo grep server.port /etc/wazuh-dashboard/opensearch_dashboards.yml
```

![Config Dashboard](../images/Pasted%20image%2020260811144908.png)

---

## 10. Ports à connaître (single-node)

| Port | Service | Commentaire |
|------|---------|-------------|
| **8443** | Dashboard | Accès UI depuis poste admin |
| **9200** | Indexer | Principalement local (Filebeat sur la même machine) |
| **55000** | API Manager | Admin / intégrations |
| **1514** tcp/udp | Agent events | Collecte agents |
| **1515** tcp | Enrollment | Enregistrement agents |
| **443** | BunkerWeb | **Ne pas toucher** |

---

## 11. Firewall local (firewalld)

```bash
sudo systemctl is-active firewalld
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --permanent --add-port=1514/udp
sudo firewall-cmd --permanent --add-port=1515/tcp
sudo firewall-cmd --permanent --add-port=55000/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

![Ports firewalld](../images/Pasted%20image%2020260811150614.png)

---

## 12. Règles OPNsense (interface **LAN**, direction **in**)

### Clarification (souvent source de confusion)

D’après le modèle OPNsense et la fiche firewall du lab (**04b**) :

- **Direction `in`** = trafic qui **entre dans le firewall** sur **cette** interface.
- Pour **poste admin (LAN) → bastion (OPT1)** : la règle se pose sur l’interface **LAN**, direction **in** — **pas** comme un “inbound OPT1” naïf.

Même logique que SSH management / services web vers l’ancien bastion.

### Règles (à placer **au-dessus** du Block All LAN)

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | Poste admin (alias) | Bastion G4 (alias) | **8443** | Wazuh Dashboard |
| Pass | Poste admin | Bastion G4 | **55000** | Wazuh API |
| Pass | Poste admin | Bastion G4 | 1514/1515 | Agents (si tests depuis le LAN) |

- Log : Yes  
- Aliases typiques : host bastion = `10.0.10.y`, ports nommés `Wazuh_Dashboard` / `Wazuh_API` / `Wazuh_Agent`

Côté **OPT1**, pas de règle “in” spécifique nécessaire pour que le Dashboard réponde : OPT1 reste centré sortie bastion + block all + anti-pivot LAN.

---

## 13. Démarrage Dashboard + validation

```bash
sudo systemctl enable --now wazuh-dashboard
sudo systemctl status wazuh-dashboard --no-pager

curl -k -I https://127.0.0.1:8443
curl -k -I https://10.0.10.y:8443
sudo ss -tulnp | grep 8443
```

Réponse attendue : **HTTP 302** → `/app/login`.

### Depuis le poste d’administration (PowerShell)

```powershell
Test-NetConnection 10.0.10.y -Port 22
Test-NetConnection 10.0.10.y -Port 8443
```

| Résultat | Lecture |
|----------|---------|
| `curl` local OK, Test-Net 8443 **KO** | Blocage **OPNsense** (règle LAN) |
| `curl` local **KO** | Dashboard down / mauvais port |
| Port 22 OK, 8443 KO | Réseau OK, **8443 filtré** |
| firewalld sans `8443/tcp` | Port pas ouvert sur Rocky |

Navigateur : `https://10.0.10.y:8443`  
(Certificat auto-signé → accepter le risque pour le lab)

![Page login Wazuh](../images/Pasted%20image%2020260812081503.png)

**Identifiants par défaut :** `admin` / `admin`  
→ à **changer immédiatement** (fiche **30a**).

---

## 14. Points d’attention post-install

| Sujet | Détail |
|-------|--------|
| RAM | Indexer + Manager + Dashboard : plusieurs Go utiles ; 16 Go min confortable en dual channel |
| Plan management | Dashboard sur IP management uniquement, source poste admin |
| VPN poste client | Peut casser l’accès local ; exclure le navigateur ou couper le temps du lab |
| Accès distant futur | **WireGuard** (pas d’exposition 8443 sur Internet) |

---

## 15. Checklist de fin d’installation

- [x] Indexer installé, security init **GREEN**
- [x] Manager + Filebeat UP, `filebeat test output` OK
- [x] Dashboard sur **8443** (pas 443), login accessible depuis le poste admin
- [x] firewalld : 8443, 1514, 1515, 55000
- [x] OPNsense LAN : règles Wazuh (direction **in**)
- [ ] Mot de passe admin changé → **30a**
- [ ] Premier agent / health checks UI → **30a**

---

## Références internes

- Fiche 23 – Install Rocky + réseau G4  
- Fiche 26 – BunkerWeb (d’où le conflit **443**)  
- Fiche 28 – Durcissement SSH  
- Fiche 29 – Fail2ban  
- Fiche 04b – Firewall rules / segmentation  
- **Prochaine :** **30a – Paramétrage initial Wazuh**

---

**Document fusionné – version portfolio / runbook – 15 août 2026**
