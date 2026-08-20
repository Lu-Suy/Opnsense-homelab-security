# 33 – Incident Boot Recovery + Journal persistant + Paramétrage BIOS

**Projet :** Horus AIS – Bastion  
**Date :** 17 août 2026  
**Machine :** Bastion G4 (Rocky Linux 10) – `bastion-godmode`  
**Objectif :** Documenter l’incident de démarrage après coupure de courant, les actions correctives et les vérifications de sécurité.

---

## 1. Problème rencontré

Après une coupure de courant, la machine G4 a redémarré et s’est retrouvée bloquée sur un écran bleu de **Boot Recovery** avec le message :

> « Press any key for load menu »

Tant que personne n’intervenait au clavier, la machine restait bloquée :
- Pas de réseau
- Pas de SSH
- Impossible de se connecter à distance

Une fois la touche pressée et le boot forcé, le système a démarré normalement et le SSH est redevenu accessible.

---

## 2. Première constatation importante

Le **journal systemd n’était pas persistant**.

Conséquences :
- Aucun log des boots précédents n’était disponible
- Impossible d’identifier précisément la cause du passage en recovery
- Difficulté à écarter une éventuelle intrusion

---

## 3. Correction : activation du journal persistant

**Commandes réalisées :**

```bash
# Création du dossier de stockage des logs
sudo mkdir -p /var/log/journal

# Configuration forçant le mode persistant
sudo mkdir -p /etc/systemd/journald.conf.d

echo -e "[Journal]\nStorage=persistent\nSystemMaxUse=500M" | sudo tee /etc/systemd/journald.conf.d/persistent.conf

# Application de la configuration
sudo systemctl restart systemd-journald
```

**Explication des commandes :**

| Commande / Paramètre              | Effet                                                              |
|-----------------------------------|--------------------------------------------------------------------|
| `mkdir -p /var/log/journal`       | Crée l’emplacement de stockage permanent des logs                  |
| `Storage=persistent`              | Force l’écriture des logs sur le disque (au lieu de la RAM)        |
| `SystemMaxUse=500M`               | Limite la taille maximale des journaux à 500 Mo                    |
| `systemctl restart systemd-journald` | Recharge le service de journalisation                           |

**Vérification :**

```bash
journalctl --disk-usage
```

Résultat observé : `Archived and active journals take up 9.6M`

Les logs survivent désormais aux redémarrages.

---

## 4. Vérifications de sécurité effectuées

En l’absence de logs historiques, une série de contrôles a été réalisée pour écarter une éventuelle intrusion.

### 4.1 Historique des connexions

```bash
sudo last -a | head -30
```

→ Uniquement des connexions depuis `10.0.0.10` (AlphaDeck). Aucune IP inconnue.

### 4.2 Logs d’authentification SSH

```bash
sudo cat /var/log/secure | grep -i "failed\|accepted" | tail -50
```

→ Uniquement des authentifications réussies avec la clé connue.

### 4.3 Clés SSH autorisées

```bash
sudo cat /root/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
```

→ Une seule clé présente (`doo@AlphaDeck-2026`) sur root et hyper_doo.

### 4.4 Liste des utilisateurs

```bash
cat /etc/passwd
```

→ Utilisateurs normaux + comptes de service (wazuh, cloudflared…). Rien de suspect.

### 4.5 Services en cours d’exécution

```bash
systemctl list-units --type=service --state=running
```

→ Services attendus (Docker, Wazuh, cloudflared, fail2ban, etc.).

### 4.6 Processus

```bash
ps aux --sort=-%cpu | head -20
```

→ Rien d’anormal.

---

## 5. Conclusion des vérifications

**Aucun signe d’intrusion n’a été détecté.**

Le comportement observé (écran de récupération de boot + besoin d’une intervention clavier) correspond davantage à un problème de **boot loader** (timeout, entrée de boot, ou échec de démarrage précédent) qu’à une attaque.

---

## 6. État actuel

| Élément                              | Statut                        |
|--------------------------------------|-------------------------------|
| Accès SSH                            | Restauré                      |
| Journal systemd                      | Persistant                    |
| Déverrouillage distant (dracut-sshd) | Toujours configuré            |
| Cause exacte du Boot Recovery        | En cours d’analyse            |
| Signes d’attaque                     | Aucun détecté                 |

---

## 7. Mise en place d’un boot non interactif

Objectif : rendre le démarrage automatique même après une coupure de courant.

### 7.1 Vérification de la configuration GRUB actuelle

```bash
cat /etc/default/grub | grep -E "TIMEOUT|DEFAULT|RECOVERY"
```

![Capture – Configuration GRUB timeout](images/Pasted%20image%2020260817014453.png)

Résultat observé :

- `GRUB_TIMEOUT=5`
- `GRUB_DISABLE_RECOVERY="true"` ← déjà activé
- `GRUB_DEFAULT=saved`

Le recovery est déjà désactivé dans la configuration. Le problème provient probablement d’un autre mécanisme (`menu_auto_hide` ou marquage du boot précédent comme échoué).

### 7.2 Vérification de l’état grubenv

```bash
sudo grub2-editenv list
sudo cat /boot/grub2/grubenv
```

![Capture – État grubenv](images/Pasted%20image%2020260817014518.png)

État observé :

- `menu_auto_hide=1`
- `boot_success=1`
- `boot_indeterminate=0`

Après la coupure de courant, le système a considéré le boot précédent comme échoué et a affiché le menu de recovery en attendant une intervention clavier.

### 7.3 Solution appliquée

Ajout du paramètre `GRUB_RECORDFAIL_TIMEOUT=0` :

```bash
echo 'GRUB_RECORDFAIL_TIMEOUT=0' | sudo tee -a /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Effet :**  
Même après une coupure de courant brutale, la machine démarre automatiquement sans attendre d’intervention clavier.

### 7.4 Vérification

```bash
grep RECORDFAIL /etc/default/grub
```

La ligne `GRUB_RECORDFAIL_TIMEOUT=0` doit apparaître.

---

## 8. État final

| Élément                              | Statut                        |
|--------------------------------------|-------------------------------|
| Accès SSH                            | Restauré                      |
| Journal systemd                      | Persistant                    |
| GRUB_RECORDFAIL_TIMEOUT              | = 0                           |
| Déverrouillage distant (dracut-sshd) | Toujours configuré            |
| Signes d’attaque                     | Aucun détecté                 |

---

**Fin de la fiche 33**
