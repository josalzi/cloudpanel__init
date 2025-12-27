# Guide Complet : Sécurisation VPS CloudPanel avec CrowdSec

## Ubuntu 24.04 + CloudPanel + CrowdSec + Tailscale

> **Temps estimé** : 45-60 minutes  
> **Niveau** : Intermédiaire  
> **Prérequis** : Accès root SSH à ton VPS Hostinger  
> **Use case** : Développement nomade depuis WiFi d'hôtels (IPs dynamiques)

---

## Ordre d'exécution

L'ordre est **crucial** pour ne pas se bloquer l'accès :

1. Tailscale sur VPS (accès de secours garanti)
2. Tailscale sur tes devices (PC + Steam Deck)
3. Clés SSH (génération + déploiement)
4. UFW via CloudPanel (ouvrir le nouveau port SSH)
5. Hardening SSH (changer le port, désactiver passwords)
6. CrowdSec (protection)

---

## Table des matières

1. [Préparation du VPS](#1-préparation-du-vps)
2. [Tailscale - Installation VPS](#2-tailscale---installation-vps)
3. [Tailscale - Installation Devices](#3-tailscale---installation-devices)
4. [Clés SSH multi-devices](#4-clés-ssh-multi-devices)
5. [UFW via CloudPanel](#5-ufw-via-cloudpanel)
6. [Hardening SSH](#6-hardening-ssh)
7. [Installation CrowdSec](#7-installation-crowdsec)
8. [Configuration CrowdSec pour CloudPanel](#8-configuration-crowdsec-pour-cloudpanel)
9. [Bouncers CrowdSec](#9-bouncers-crowdsec)
10. [Collections et Scénarios](#10-collections-et-scénarios)
11. [CrowdSec Console](#11-crowdsec-console)
12. [Whitelisting Tailscale](#12-whitelisting-tailscale)
13. [Mises à jour automatiques](#13-mises-à-jour-automatiques)
14. [Commandes de maintenance](#14-commandes-de-maintenance)
15. [Tests et validation](#15-tests-et-validation)
16. [Workflow nomade](#16-workflow-nomade)

---

## 1. Préparation du VPS

Connecte-toi à ton VPS via SSH (méthode actuelle) :

```bash
ssh root@ton-ip-publique-vps
```

### 1.1 Mise à jour du système

```bash
apt update && apt upgrade -y
apt install curl gnupg lsb-release software-properties-common -y
```

### 1.2 Vérification de la synchronisation horaire

CrowdSec dépend de timestamps précis :

```bash
timedatectl set-ntp true
timedatectl status
```

Tu devrais voir `NTP service: active`.

### 1.3 Sauvegarder la config SSH actuelle

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

---

## 2. Tailscale - Installation VPS

> ⚠️ On installe Tailscale EN PREMIER pour avoir un accès de secours si on se bloque avec SSH/UFW.

### 2.1 Créer un compte Tailscale

1. Va sur [https://tailscale.com](https://tailscale.com)
2. Crée un compte (GitHub, Google, ou email)
3. Note ton "Tailnet name" (ex: `joey.ts.net`)

### 2.2 Installation sur le VPS

```bash
# Installer Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Démarrer et authentifier
tailscale up
```

Un lien d'authentification s'affiche. Ouvre-le dans ton navigateur et autorise le VPS.

### 2.3 Vérifier et noter l'IP Tailscale du VPS

```bash
tailscale ip -4
```

**Note cette IP** (ex: `100.100.100.30`), tu en auras besoin pour la config SSH.

### 2.4 Désactiver l'expiration de clé (important pour un serveur)

1. Va sur [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)
2. Trouve ton VPS dans la liste
3. Clique sur le menu `...` → **Disable key expiry**

### 2.5 Vérifier que Tailscale fonctionne

```bash
tailscale status
```

Tu devrais voir ton VPS listé comme "online".

---

## 3. Tailscale - Installation Devices

### 3.1 Sur Windows (PC Laragon)

1. Télécharge l'installer : [https://tailscale.com/download/windows](https://tailscale.com/download/windows)
2. Installe et connecte-toi avec le **même compte** que le VPS
3. L'icône Tailscale apparaît dans la barre des tâches

**Vérifier l'IP Tailscale Windows** (PowerShell) :

```powershell
tailscale ip -4
```

Note cette IP (ex: `100.100.100.10`).

### 3.2 Sur Steam Deck (ArchLinux)

En mode Desktop, ouvre Konsole :

```bash
# Désactiver le read-only filesystem temporairement
sudo steamos-readonly disable

# Initialiser pacman (si première fois)
sudo pacman-key --init
sudo pacman-key --populate archlinux

# Installer Tailscale
sudo pacman -S tailscale

# Activer et démarrer le service
sudo systemctl enable --now tailscaled

# Se connecter (même compte que VPS et PC)
sudo tailscale up

# Réactiver le read-only
sudo steamos-readonly enable
```

**Vérifier l'IP Tailscale Steam Deck** :

```bash
tailscale ip -4
```

Note cette IP (ex: `100.100.100.20`).

### 3.3 Tester la connectivité Tailscale

Depuis ton PC Windows (PowerShell) ou Steam Deck :

```bash
# Ping le VPS via son IP Tailscale
ping 100.100.100.30
```

Si ça répond, Tailscale fonctionne. Tu as maintenant un accès de secours garanti.

---

## 4. Clés SSH multi-devices

### 4.1 Pourquoi une clé par device ?

- **Révocation indépendante** : Si tu perds ton Steam Deck, tu révoques uniquement sa clé
- **Traçabilité** : Tu sais quel device s'est connecté
- **Sécurité** : Pas de copie de clé privée entre machines

### 4.2 Générer la clé sur Windows (PC Laragon)

Ouvre PowerShell :

```powershell
# Créer le dossier .ssh s'il n'existe pas
mkdir -Force $env:USERPROFILE\.ssh

# Générer la clé (sans passphrase : appuie Enter deux fois)
ssh-keygen -t ed25519 -C "joey-pc-windows" -f $env:USERPROFILE\.ssh\id_ed25519_vps
```

Cela crée :
- `C:\Users\TonUsername\.ssh\id_ed25519_vps` (clé privée)
- `C:\Users\TonUsername\.ssh\id_ed25519_vps.pub` (clé publique)

**Afficher la clé publique pour la copier** :

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519_vps.pub
```

Copie le contenu affiché (commence par `ssh-ed25519`).

### 4.3 Générer la clé sur Steam Deck

En mode Desktop, ouvre Konsole :

```bash
# Créer le dossier .ssh s'il n'existe pas
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Générer la clé (sans passphrase : appuie Enter deux fois)
ssh-keygen -t ed25519 -C "joey-steamdeck" -f ~/.ssh/id_ed25519_vps
```

**Afficher la clé publique** :

```bash
cat ~/.ssh/id_ed25519_vps.pub
```

Copie le contenu.

### 4.4 Ajouter les clés publiques sur le VPS

Sur le VPS (connecté via SSH classique pour l'instant) :

```bash
# Éditer le fichier authorized_keys
nano ~/.ssh/authorized_keys
```

Ajoute les deux clés publiques (une par ligne) :

```
ssh-ed25519 AAAA...clé-complete... joey-pc-windows
ssh-ed25519 AAAA...clé-complete... joey-steamdeck
```

Sauvegarde (`Ctrl+O`, `Enter`, `Ctrl+X`).

**Corriger les permissions** :

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 4.5 Configurer le fichier SSH config sur Windows

Crée/édite le fichier `C:\Users\TonUsername\.ssh\config` :

```powershell
notepad $env:USERPROFILE\.ssh\config
```

Contenu :

```
# Connexion via Tailscale (utiliser celle-ci)
Host vps
    HostName 100.100.100.30
    Port 22
    User root
    IdentityFile ~/.ssh/id_ed25519_vps

# Connexion de secours via IP publique
Host vps-public
    HostName 203.0.113.50
    Port 22
    User root
    IdentityFile ~/.ssh/id_ed25519_vps
```

> **Note** : Remplace `100.100.100.30` par l'IP Tailscale de ton VPS et `203.0.113.50` par son IP publique Hostinger.
> 
> **Note 2** : Le port est encore 22. On le changera après avoir configuré UFW.

### 4.6 Configurer le fichier SSH config sur Steam Deck

```bash
nano ~/.ssh/config
```

Même contenu :

```
# Connexion via Tailscale (utiliser celle-ci)
Host vps
    HostName 100.100.100.30
    Port 22
    User root
    IdentityFile ~/.ssh/id_ed25519_vps

# Connexion de secours via IP publique
Host vps-public
    HostName 203.0.113.50
    Port 22
    User root
    IdentityFile ~/.ssh/id_ed25519_vps
```

**Permissions** :

```bash
chmod 600 ~/.ssh/config
```

### 4.7 Tester la connexion avec clé SSH

Depuis Windows (PowerShell) :

```powershell
ssh vps
```

Depuis Steam Deck :

```bash
ssh vps
```

Tu devrais te connecter **sans mot de passe** (grâce à la clé SSH).

> ⚠️ Si ça ne fonctionne pas, vérifie que les clés publiques sont bien dans `authorized_keys` sur le VPS.

---

## 5. UFW via CloudPanel

### 5.1 Accéder à l'interface Firewall

1. Connecte-toi à CloudPanel : `https://ton-ip-publique:8443`
2. Va dans **Admin Area** → **Security** → **Firewall**

### 5.2 Ajouter les nouvelles règles

Clique sur **Add Rule** pour chaque règle :

| Port Range | Source | Description |
|------------|--------|-------------|
| `2222` | `0.0.0.0/0` | SSH Custom Port |
| `80` | `0.0.0.0/0` | HTTP |
| `443` | `0.0.0.0/0` | HTTPS |
| `8443` | `100.64.0.0/10` | CloudPanel Tailscale Only |

> 💡 La règle `8443` avec source `100.64.0.0/10` signifie que CloudPanel ne sera accessible **que via Tailscale**.

### 5.3 Modifier/Supprimer les anciennes règles

- **Garde** la règle `22` avec `0.0.0.0/0` pour l'instant (on la supprimera après avoir testé le port 2222)
- **Supprime** la règle `8443` avec `0.0.0.0/0` si elle existe (remplacée par notre règle Tailscale)

### 5.4 Ajouter la règle Tailscale interface (CLI)

L'interface CloudPanel ne permet pas de créer des règles sur une interface réseau. Connecte-toi au VPS :

```bash
ssh vps
```

Puis :

```bash
# Autoriser tout le trafic depuis l'interface Tailscale
ufw allow in on tailscale0 comment 'Tailscale Interface'

# Vérifier
ufw status | grep tailscale
```

### 5.5 Protéger la règle Tailscale avec un script

CloudPanel peut écraser les règles CLI. Créons un script de protection :

```bash
nano /usr/local/bin/ensure-tailscale-ufw.sh
```

Contenu :

```bash
#!/bin/bash
# Ensure Tailscale UFW rule exists

if ! ufw status | grep -q "tailscale0"; then
    ufw allow in on tailscale0 comment 'Tailscale Interface'
    echo "$(date): Tailscale UFW rule restored" >> /var/log/tailscale-ufw.log
fi
```

Rendre exécutable et ajouter au cron :

```bash
chmod +x /usr/local/bin/ensure-tailscale-ufw.sh

# Ajouter au cron
crontab -e
```

Ajoute ces lignes :

```
@reboot /usr/local/bin/ensure-tailscale-ufw.sh
0 * * * * /usr/local/bin/ensure-tailscale-ufw.sh
```

### 5.6 Vérifier la config UFW

```bash
ufw status numbered
```

Tu devrais voir le port 2222 ouvert.

---

## 6. Hardening SSH

> ⚠️ **IMPORTANT** : Garde ta session SSH actuelle ouverte pendant toute cette section !

### 6.1 Comprendre SSH sur Ubuntu 24.04

Ubuntu 24.04 utilise **socket activation** (`ssh.socket`) qui contrôle le port d'écoute. Le fichier `sshd_config` seul ne suffit plus pour changer le port !

```
ssh.socket    → Contrôle le PORT d'écoute
sshd_config   → Contrôle les OPTIONS (auth, clés, etc.)
```

### 6.2 Changer le port SSH (via ssh.socket)

```bash
# Créer un override pour ssh.socket
systemctl edit ssh.socket
```

Un éditeur s'ouvre. Ajoute ce contenu **entre les lignes de commentaires** :

```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:2222
ListenStream=[::]:2222
```

> ⚠️ Le premier `ListenStream=` vide est **obligatoire** pour effacer la valeur par défaut (22).
> 
> ⚠️ Les deux lignes suivantes sont nécessaires pour écouter en IPv4 ET IPv6.

Sauvegarde et quitte (`Ctrl+O`, `Enter`, `Ctrl+X` si nano).

### 6.3 Modifier les options SSH (sshd_config)

```bash
nano /etc/ssh/sshd_config
```

Modifie/ajoute ces lignes :

```bash
# Le port est géré par ssh.socket, mais on le met ici aussi pour cohérence
Port 2222

# Désactiver l'accès root par mot de passe (clé uniquement)
PermitRootLogin prohibit-password

# Désactiver l'authentification par mot de passe
PasswordAuthentication no

# Autres paramètres de sécurité
MaxAuthTries 3
MaxSessions 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Désactiver les méthodes non utilisées
KbdInteractiveAuthentication no
X11Forwarding no
```

### 6.4 Tester la syntaxe sshd_config

```bash
sshd -t
```

Si aucune erreur ne s'affiche, la config est valide.

### 6.5 Appliquer les changements

```bash
# Recharger systemd
systemctl daemon-reload

# Redémarrer le socket ET le service
systemctl restart ssh.socket
systemctl restart ssh
```

### 6.6 Vérifier que SSH écoute sur le bon port

```bash
ss -tlnp | grep ssh
```

**Résultat attendu** (les deux lignes sont importantes) :

```
LISTEN 0  4096   0.0.0.0:2222   0.0.0.0:*  users:(("sshd",...))
LISTEN 0  4096      [::]:2222      [::]:*  users:(("sshd",...))
```

> ⚠️ Si tu ne vois que la ligne `[::]` (IPv6), SSH n'écoutera pas sur Tailscale (qui utilise IPv4). Vérifie que tu as bien les deux `ListenStream` dans le socket override.

### 6.7 Tester la connexion (NOUVELLE session)

**Ne ferme pas ta session actuelle !**

D'abord, mets à jour le fichier config SSH sur tes devices pour utiliser le nouveau port.

**Windows** - Édite `C:\Users\TonUsername\.ssh\config` :

```
Host vps
    HostName 100.100.100.30
    Port 2222
    User root
    IdentityFile C:/Users/TonUsername/.ssh/id_ed25519_vps

Host vps-public
    HostName 203.0.113.50
    Port 2222
    User root
    IdentityFile C:/Users/TonUsername/.ssh/id_ed25519_vps
```

**Steam Deck** - Édite `~/.ssh/config` :

```bash
nano ~/.ssh/config
```

```
Host vps
    HostName 100.100.100.30
    Port 2222
    User root
    IdentityFile ~/.ssh/id_ed25519_vps

Host vps-public
    HostName 203.0.113.50
    Port 2222
    User root
    IdentityFile ~/.ssh/id_ed25519_vps
```

Puis teste depuis un **nouveau terminal** :

```bash
ssh vps
```

### 6.8 Supprimer l'ancien port 22 de UFW

Une fois que la connexion sur le nouveau port fonctionne :

1. Va dans CloudPanel → **Admin Area** → **Security** → **Firewall**
2. **Supprime** la règle `22` avec source `0.0.0.0/0`

Le port 22 reste accessible via Tailscale grâce à la règle `tailscale0`.

### 6.9 Dépannage

**"Connection refused"** après changement de port :

```bash
# Vérifier que SSH écoute
ss -tlnp | grep ssh

# Si vide ou mauvais port, vérifier l'override socket
cat /etc/systemd/system/ssh.socket.d/override.conf

# Redémarrer proprement
systemctl daemon-reload
systemctl restart ssh.socket ssh
```

**SSH écoute seulement en IPv6** (`[::]:2222` mais pas `0.0.0.0:2222`) :

Le fichier override est incomplet. Refais l'étape 6.2 avec les deux lignes `ListenStream`.

---

## 7. Installation CrowdSec

### 7.1 Ajouter le dépôt officiel

> ⚠️ Ubuntu 24.04 a une version obsolète (1.4.6) dans ses dépôts. On utilise le dépôt officiel.

```bash
curl -s https://install.crowdsec.net | sh
```

### 7.2 Configurer le pinning APT

```bash
nano /etc/apt/preferences.d/crowdsec
```

Contenu :

```
Package: *
Pin: release o=packagecloud.io/crowdsec/crowdsec,a=any,n=any,c=main
Pin-Priority: 1001
```

### 7.3 Installer CrowdSec

```bash
apt update

# Vérifier la version
apt-cache policy crowdsec

# Installer
apt install crowdsec -y
```

### 7.4 Résoudre le conflit de port (CloudPanel)

> ⚠️ **Problème fréquent** : CrowdSec utilise le port 8080 par défaut, mais CloudPanel/Nginx l'utilise déjà (Varnish/reverse proxy). Tu verras cette erreur :
> ```
> FATAL local API server stopped with error: listening on 127.0.0.1:8080: bind: address already in use
> ```

**Vérifier si le port 8080 est utilisé** :

```bash
ss -tlnp | grep 8080
```

Si Nginx apparaît, il faut changer le port de CrowdSec.

**Changer le port de l'API CrowdSec** (on utilise 6969) :

```bash
nano /etc/crowdsec/config.yaml
```

Trouve la section `api` → `server` et modifie :

```yaml
api:
  server:
    listen_uri: 127.0.0.1:6969
```

**Mettre à jour les credentials du client** :

```bash
nano /etc/crowdsec/local_api_credentials.yaml
```

Change l'URL :

```yaml
url: http://127.0.0.1:6969/
```

### 7.5 Démarrer CrowdSec

```bash
systemctl restart crowdsec
systemctl status crowdsec
```

Tu devrais voir `Active: active (running)`.

### 7.6 Vérifier l'installation

```bash
cscli version
cscli metrics
```

---

## 8. Configuration CrowdSec pour CloudPanel

### 8.1 Structure des logs CloudPanel

CloudPanel organise les logs ainsi :

```
/home/{site-user}/logs/
├── nginx/
│   ├── access.log
│   └── error.log
└── php/
    └── error.log

/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/auth.log
```

### 8.2 Configurer l'acquisition des logs

```bash
nano /etc/crowdsec/acquis.yaml
```

Remplace le contenu par :

```yaml
#-----------------------------------------
# SSH / Auth logs
#-----------------------------------------
filenames:
  - /var/log/auth.log
  - /var/log/syslog
labels:
  type: syslog
---
#-----------------------------------------
# NGINX Global logs
#-----------------------------------------
filenames:
  - /var/log/nginx/access.log
  - /var/log/nginx/error.log
labels:
  type: nginx
---
#-----------------------------------------
# CloudPanel - All sites NGINX logs
#-----------------------------------------
filenames:
  - /home/*/logs/nginx/access.log
  - /home/*/logs/nginx/error.log
labels:
  type: nginx
---
#-----------------------------------------
# Journald for SSH
#-----------------------------------------
journalctl_filter:
  - "_SYSTEMD_UNIT=ssh.service"
labels:
  type: syslog
```

### 8.3 Redémarrer CrowdSec

```bash
systemctl restart crowdsec
systemctl status crowdsec
```

---

## 9. Bouncers CrowdSec

### 9.1 Firewall Bouncer (nftables)

```bash
apt install crowdsec-firewall-bouncer-nftables -y
systemctl enable --now crowdsec-firewall-bouncer
```

Vérifier :

```bash
systemctl status crowdsec-firewall-bouncer
cscli bouncers list
```

---

## 10. Collections et Scénarios

### 10.1 Installer les collections essentielles

```bash
# Linux (SSH, système)
cscli collections install crowdsecurity/linux

# Nginx
cscli collections install crowdsecurity/nginx

# HTTP générique
cscli collections install crowdsecurity/base-http-scenarios

# Whitelists
cscli parsers install crowdsecurity/whitelists
```

### 10.2 Recharger CrowdSec

```bash
systemctl reload crowdsec
```

### 10.3 Vérifier les installations

```bash
cscli collections list
cscli scenarios list
```

---

## 11. CrowdSec Console

### 11.1 Créer un compte

1. Va sur [https://app.crowdsec.net](https://app.crowdsec.net)
2. Crée un compte gratuit
3. Dans "Security Engines", clique sur "Add Security Engine"
4. Copie la clé d'enrollment

### 11.2 Enroller le VPS

```bash
cscli console enroll <TA-CLE-DENROLLMENT>
```

### 11.3 Valider sur la Console web

Retourne sur la Console et accepte la demande d'enrollment.

### 11.4 Activer le partage communautaire

```bash
cscli console enable -a
systemctl reload crowdsec
```

---

## 12. Whitelisting Tailscale

### 12.1 Créer le fichier de whitelist

```bash
nano /etc/crowdsec/parsers/s02-enrich/my-whitelists.yaml
```

Contenu :

```yaml
name: crowdsecurity/my-whitelists
description: "Whitelist for Tailscale network"
whitelist:
  reason: "Tailscale network - always trusted"
  cidr:
    - "100.64.0.0/10"
    - "10.0.0.0/8"
    - "172.16.0.0/12"
    - "192.168.0.0/16"
    - "127.0.0.0/8"
```

### 12.2 Appliquer

```bash
systemctl reload crowdsec

# Vérifier
cscli parsers list | grep whitelist
```

---

## 13. Mises à jour automatiques

### 13.1 Mises à jour Ubuntu

```bash
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
```

Choisis "Yes".

### 13.2 Mises à jour CrowdSec Hub

```bash
crontab -e
```

Ajoute :

```bash
# Mise à jour hub CrowdSec tous les jours à 4h
0 4 * * * /usr/bin/cscli hub update && /usr/bin/cscli hub upgrade --all
```

---

## 14. Commandes de maintenance

### 14.1 Monitoring

```bash
# Métriques globales
cscli metrics

# Alertes récentes
cscli alerts list --since 24h

# IPs bannies
cscli decisions list

# Statut services
systemctl status crowdsec
systemctl status crowdsec-firewall-bouncer
```

### 14.2 Gestion des bans

```bash
# Bannir une IP (4h)
cscli decisions add -i 1.2.3.4 -t ban -d 4h -r "Manual ban"

# Débannir
cscli decisions delete -i 1.2.3.4

# Tout débannir
cscli decisions delete --all
```

### 14.3 Mise à jour manuelle du hub

```bash
cscli hub update
cscli hub upgrade --all
systemctl reload crowdsec
```

---

## 15. Tests et validation

### 15.1 Tester le firewall bouncer

Depuis une machine NON connectée à Tailscale, fais plusieurs tentatives SSH échouées :

```bash
for i in {1..10}; do ssh -p 2222 fakeuser@ip-publique-vps; done
```

Sur le VPS :

```bash
cscli decisions list
```

L'IP devrait être bannie.

### 15.2 Vérifier que Tailscale n'est pas banni

Depuis ton PC ou Steam Deck (connecté à Tailscale) :

```bash
ssh vps
```

Ça doit fonctionner même après le test précédent.

### 15.3 Vérifier les métriques

```bash
cscli metrics
```

Tu devrais voir des lignes parsées pour nginx et syslog.

---

## 16. Workflow nomade

### 16.1 Checklist nouvelle escale

```bash
# 1. Connecte-toi au WiFi de l'hôtel

# 2. Vérifie que Tailscale est connecté
tailscale status

# 3. Travaille !
ssh vps
```

C'est tout. Pas de configuration, pas de whitelist à ajouter.

### 16.2 En cas de problème

| Problème | Solution |
|----------|----------|
| Tailscale ne se connecte pas | `sudo systemctl restart tailscaled` |
| SSH timeout | `tailscale ping 100.x.x.30` pour tester |
| Connection refused | VPS peut-être reboot, attends 2-3 min |
| Banni par erreur | Console VNC Hostinger → `cscli decisions delete --all` |

### 16.3 Révocation d'urgence (perte d'un device)

Si tu perds ton Steam Deck :

1. Sur le VPS : `nano ~/.ssh/authorized_keys` → Supprime la ligne `joey-steamdeck`
2. Sur Tailscale : [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines) → Supprime le Steam Deck

---

## Récapitulatif architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         INTERNET                                │
│                                                                 │
│    ┌──────────────┐         ┌──────────────┐                   │
│    │ Steam Deck   │         │  PC Windows  │                   │
│    │ WiFi Hôtel   │         │  Domicile    │                   │
│    └──────┬───────┘         └──────┬───────┘                   │
│           │                        │                           │
│           └────────┬───────────────┘                           │
│                    │                                           │
│           ┌────────▼────────┐                                  │
│           │   TAILSCALE     │                                  │
│           │  100.64.0.0/10  │                                  │
│           └────────┬────────┘                                  │
└────────────────────┼───────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VPS HOSTINGER                                │
├─────────────────────────────────────────────────────────────────┤
│  UFW (CloudPanel GUI + script Tailscale)                       │
│      → 80, 443, 2222 : publics                                 │
│      → 8443 : Tailscale only                                   │
│      → tailscale0 : tout autorisé                              │
├─────────────────────────────────────────────────────────────────┤
│  CrowdSec                                                       │
│      → 100.64.0.0/10 whitelisté                                │
│      → Autres IPs : analyse + ban                              │
├─────────────────────────────────────────────────────────────────┤
│  CloudPanel + NGINX + PHP-FPM                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Checklist finale

### Serveur
- [ ] Tailscale installé et "Disable key expiry" activé
- [ ] CrowdSec installé depuis dépôt officiel
- [ ] Firewall bouncer nftables actif
- [ ] acquis.yaml configuré pour `/home/*/logs/nginx/*.log`
- [ ] Whitelist `100.64.0.0/10` dans CrowdSec
- [ ] SSH sur port 2222, password auth désactivé
- [ ] UFW : 8443 limité à `100.64.0.0/10`
- [ ] Script `ensure-tailscale-ufw.sh` dans cron

### Devices
- [ ] Tailscale sur PC Windows
- [ ] Tailscale sur Steam Deck
- [ ] Clé SSH `joey-pc-windows` générée et déployée
- [ ] Clé SSH `joey-steamdeck` générée et déployée
- [ ] Fichier `~/.ssh/config` configuré (port 2222)

### Tests
- [ ] `ssh vps` fonctionne depuis PC
- [ ] `ssh vps` fonctionne depuis Steam Deck
- [ ] CloudPanel accessible via `https://100.x.x.30:8443`
- [ ] IP externe bannie après brute-force test
- [ ] IP Tailscale jamais bannie

---

*Guide v2 - Corrigé pour cohérence et commandes Windows*
