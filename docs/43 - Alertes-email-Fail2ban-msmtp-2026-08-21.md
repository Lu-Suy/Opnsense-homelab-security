# 43 – Alertes email (Fail2ban + msmtp) – G4

**Date :** 21 août 2026 (soir)  
**Machine :** HP EliteDesk 800 G4 (Rocky Linux 10) – hostname `bastion-godmode`  
**Objectif :** Mettre en place un canal d’alertes email fiable depuis le bastion vers `alertes@horus-ais.com` (alias Zimbra OVH), en commençant par Fail2ban.  
**Statut :** Validé (mail de test + alerte Fail2ban reçus)

---
## 1. Contexte et objectif

Jusqu’à présent, le lab disposait d’une détection (Fail2ban, Wazuh, OPNsense) mais **aucune notification proactive** par email.

Pour rendre le lab présentable et opérationnel (démo recruteurs, monitoring quotidien), il fallait :

1. Créer une adresse d’alerte professionnelle (`alertes@horus-ais.com`)
2. Configurer un MTA léger capable d’envoyer des mails depuis la G4
3. Brancher Fail2ban sur ce canal
4. Valider de bout en bout

**Choix technique retenu :**  
- Alias Zimbra (gratuit) plutôt qu’une nouvelle boîte payante  
- **msmtp** (léger) plutôt que Postfix complet en mode satellite

---

## 2. Préparation côté messagerie (OVH Zimbra)

### 2.1 Création de l’alias

Sur l’espace client OVH → service Zimbra :

1. Onglet **Comptes e-mail**
2. Menu ⋮ de la boîte existante → **Modifier**
3. Onglet **Alias** → **Créer un Alias**
4. Alias créé : `alertes@horus-ais.com` → redirection vers la boîte principale

**Avantage :** aucun coût supplémentaire, adresse professionnelle, tous les mails arrivent dans la même boîte.

---

## 3. État initial de Fail2ban

```bash
sudo systemctl status fail2ban
# active (running)

sudo fail2ban-client status
# Jail list: sshd

sudo grep -E "destemail|sender|mta|action" /etc/fail2ban/jail.local /etc/fail2ban/jail.conf 2>/dev/null | head -20
# destemail = root@localhost
```

Fail2ban fonctionnait, mais envoyait les mails vers `root@localhost` (aucune livraison externe).

---

## 4. Configuration de Fail2ban vers alertes@

Édition de `/etc/fail2ban/jail.local` :

```ini
[DEFAULT]
destemail = alertes@horus-ais.com
sender = fail2ban@horus-ais.com
mta = sendmail
action = %(action_mwl)s
```

```bash
sudo fail2ban-client reload
sudo systemctl restart fail2ban

sudo fail2ban-client get sshd actions
sudo grep -E "destemail|sender|action" /etc/fail2ban/jail.local
```

![[Pasted image 20260821165811.png]]

À ce stade, Fail2ban était prêt côté configuration, mais le MTA local n’était pas capable d’envoyer vers l’extérieur.


---

## 5. Diagnostic MTA

```bash
which postfix
# not found

which sendmail
# /usr/sbin/sendmail

systemctl status sendmail 2>/dev/null || echo "sendmail non actif"
# sendmail non actif

mailq 2>/dev/null || echo "pas de mailq"
```

**Conclusion :**  
Le binaire `sendmail` existe (compatibilité), mais aucun MTA n’est réellement configuré pour livrer des mails vers Internet. Fail2ban ne pouvait donc pas envoyer d’alertes.

---

## 6. Installation de msmtp

### Pourquoi msmtp plutôt que Postfix ?

| Critère              | msmtp                          | Postfix (satellite)       |
|----------------------|--------------------------------|-----------------------------|
| Objectif             | Envoyer des alertes uniquement | Serveur mail complet        |
| Complexité           | Faible                         | Moyenne                     |
| Ressources           | Très léger                     | Plus lourd                  |
| Temps d’installation | 15-25 min                      | 30-45 min                   |
| Adapté au lab        | Excellent                      | Overkill                    |

**Décision :** msmtp.

```bash
sudo dnf install -y msmtp
```

*(Le sous-paquet `msmtp-mta` n’était pas disponible dans les dépôts Rocky 10 / EPEL au moment de l’installation.)*

---

## 7. Configuration de msmtp

Fichier `/etc/msmtprc` :

```ini
# Configuration msmtp pour alertes
defaults
auth           on
tls            on
tls_trust_file /etc/pki/tls/certs/ca-bundle.crt
logfile        /var/log/msmtp.log

account        ovh
host           ssl0.ovh.net
port           587
from           alertes@horus-ais.com
user           <adresse_zimbra_principale>
password       <mot_de_passe>

account default : ovh
```

```bash
sudo chmod 600 /etc/msmtprc
sudo chown root:root /etc/msmtprc
sudo touch /var/log/msmtp.log
sudo chmod 666 /var/log/msmtp.log
```

### Note de sécurité (point d’amélioration)

Le mot de passe est stocké en clair dans `/etc/msmtprc` (permissions 600, root uniquement).  

**C’est la pratique courante** pour les alertes système sur un serveur de lab, mais ce n’est **pas idéal** en environnement professionnel.

**Améliorations possibles plus tard :**
- Fichier de mot de passe séparé avec permissions encore plus strictes
- Utilisation de `passwordeval` (commande qui fournit le mot de passe)
- Intégration avec un coffre de secrets (Vault, systemd-creds, etc.)

Pour l’instant, la solution est acceptable et documentée comme point d’attentio

---

## 8. Problème de connectivité (port 587)

Premier test :

```bash
echo "Test alerte depuis la G4 via msmtp" | sudo msmtp --file=/etc/msmtprc -a ovh alertes@horus-ais.com
```
![[Pasted image 20260821173128.png]]
→ Commande bloquée (timeout). Log vide.

**Cause identifiée :**  
Le port **587/TCP** (SMTP submission STARTTLS) n’était pas autorisé en sortie depuis OPT1.

### Règle ajoutée

| Champ            | Valeur                          |
| ---------------- | ------------------------------- |
| Action           | **Pass**                        |
| Interface        | OPT1                            |
| Direction        | in (ou out selon ton habitude)  |
| Protocol         | TCP                             |
| Source           | OPT1 net / OPT1_Bastion network |
| Destination      | any                             |
| Destination port | **587**                         |
| Description      | Allow SMTP submission (msmtp)   |
| Log              | coché (pour debug)              |
**Note :** La G4 possède deux adresses IP (`.10` services / `.11` management). L’utilisation de `OPT1 net` comme source couvre les deux.

---

## 9. Validation de l’envoi msmtp

```bash
echo "Test alerte depuis la G4 via msmtp" | sudo msmtp --file=/etc/msmtprc -a ovh alertes@horus-ais.com
sudo tail -10 /var/log/msmtp.log
```

![[Pasted image 20260821174407.png]]
**Log :**

```
host=ssl0.ovh.net tls=on auth=on ... smtpstatus=250 smtpmsg='250 2.0.0 Ok: queued as D3751781AFD' exitcode=EX_OK
```


![[Pasted image 20260821174449.png]]

Mail de test bien reçu.

---

## 10. Branchement de Fail2ban sur msmtp

```bash
ls -l /usr/sbin/sendmail
# → /usr/sbin/sendmail → /etc/alternatives/mta

ls -l /etc/alternatives/mta
# → /etc/alternatives/mta → /usr/bin/msmtp
```

![[Pasted image 20260821175223.png]]

**Explication :**  
Lors de l’installation du paquet `msmtp`, celui-ci s’est automatiquement enregistré auprès du système d’alternatives de Rocky Linux et s’est positionné comme fournisseur de `mta`.  
Aucun lien symbolique n’a été créé manuellement.

Fail2ban, qui appelle `sendmail`, passe donc automatiquement par msmtp.
### Test final Fail2ban

```bash
sudo fail2ban-client reload
sudo fail2ban-client set sshd banip 203.0.113.50
```

![[Pasted image 20260821175444.png]]

Mail d’alerte reçu :

```
[Fail2Ban] sshd: banned 203.0.113.50 from bastion-godmode
```

Deban :

```bash
sudo fail2ban-client set sshd unbanip 203.0.113.50
```

---

## 11. Synthèse

| Élément | Statut | Détail |
|---------|--------|--------|
| Alias `alertes@horus-ais.com` | ✅ | Créé sur OVH Zimbra |
| msmtp installé + configuré | ✅ | `/etc/msmtprc` |
| Règle firewall OPT1 port 587 | ✅ | Source = OPT1 net |
| Test envoi msmtp | ✅ | smtpstatus=250 |
| Fail2ban → alertes@ | ✅ | Alerte reçue |
| Lien sendmail → msmtp | ✅ | Via alternatives |

---

## 12. Points d’attention

1. **Mot de passe en clair** dans `/etc/msmtprc` → à améliorer (fichier séparé ou `passwordeval`).
2. **whois manquant** → installer le paquet `whois` pour enrichir les mails Fail2ban.
3. **Autres sources d’alertes** à brancher ensuite :
   - Wazuh (alertes de détection)
   - OPNsense (notifications firewall / Suricata)
4. **SPF / DKIM** : vérifier que l’envoi depuis la G4 via le compte Zimbra reste bien aligné avec les enregistrements DNS du domaine.
5. **Rate-limit / anti-spam** côté réception si le volume d’alertes augmente.


---
## 13. Commandes de référence rapides

```bash
# Test d’envoi manuel
echo "Test" | sudo msmtp --file=/etc/msmtprc -a ovh alertes@horus-ais.com

# Log msmtp
sudo tail -f /var/log/msmtp.log

# Test ban Fail2ban
sudo fail2ban-client set sshd banip 203.0.113.50
sudo fail2ban-client set sshd unbanip 203.0.113.50

# Vérifier les actions de la jail
sudo fail2ban-client get sshd actions
```

---

**Fin de la fiche 43**  
Canal d’alertes email opérationnel depuis le bastion G4 (Fail2ban → msmtp → OVH Zimbra).