# Guide Complet : Sécurisation VPS CloudPanel avec CrowdSec

## Ubuntu 24.04 + CloudPanel + CrowdSec + UFW + Tailscale

> **Temps estimé** : 45-60 minutes  
> **Niveau** : Intermédiaire  
> **Prérequis** : Accès root SSH à ton VPS Hostinger  
> **Use case** : Développement nomade depuis WiFi d'hôtels (IPs dynamiques)

---

## Table des matières

1. [Préparation et prérequis](#1-préparation-et-prérequis)
2. [Tailscale - VPN Mesh Zero-Config](#2-tailscale---vpn-mesh-zero-config)
3. [Gestion des clés SSH multi-devices](#3-gestion-des-clés-ssh-multi-devices)
4. [Hardening SSH](#4-hardening-ssh)
5. [Configuration UFW](#5-configuration-ufw)
6. [Installation CrowdSec](#6-installation-crowdsec)
7. [Configuration pour CloudPanel](#7-configuration-pour-cloudpanel)
8. [Installation des Bouncers](#8-installation-des-bouncers)
9. [Collections et Scénarios](#9-collections-et-scénarios)
10. [CrowdSec Console](#10-crowdsec-console)
11. [Whitelisting intelligent](#11-whitelisting-intelligent)
12. [Mises à jour automatiques](#12-mises-à-jour-automatiques)
13. [Commandes de maintenance](#13-commandes-de-maintenance)
14. [Tests et validation](#14-tests-et-validation)
15. [Workflow nomade - Checklist escale](#15-workflow-nomade---checklist-escale)

---

## 1. Préparation et prérequis

### 1.1 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl gnupg lsb-release software-properties-common -y
```

### 1.2 Vérification de la synchronisation horaire

CrowdSec dépend de timestamps précis :

```bash
sudo timedatectl set-ntp true
timedatectl status
```

Tu devrais voir `NTP service: active`.

### 1.3 Backup de la configuration SSH actuelle

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

### 1.4 Note importante sur les ports CloudPanel

CloudPanel utilise les ports suivants :
- **8443** : Interface web CloudPanel
- **22** : SSH (que nous allons changer)
- **80/443** : HTTP/HTTPS
- **3306** : MariaDB (localhost uniquement)

---

## 2. Tailscale - VPN Mesh Zero-Config

### Pourquoi Tailscale ?

En tant que pilote, tu te connectes depuis des WiFi d'hôtels avec des IPs qui changent à chaque escale. Impossible de whitelister ces IPs. Tailscale résout ce problème :

- **IP stable** : Ton Steam Deck aura toujours la même IP Tailscale (ex: `100.x.x.x`)
- **Zero-config** : Une fois installé, ça marche. Pas de reconfiguration à chaque hôtel
- **Traverse les NAT** : Fonctionne même derrière les WiFi restrictifs d'hôtels
- **Chiffrement WireGuard** : Sécurisé de bout en bout
- **Gratuit** : Jusqu'à 100 devices pour usage personnel

### 2.1 Architecture cible

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   PC Windows    │     │   Steam Deck    │     │   VPS Hostinger │
│   (Laragon)     │     │   (ArchLinux)   │     │   (CloudPanel)  │
│                 │     │                 │     │                 │
│ Tailscale IP:   │     │ Tailscale IP:   │     │ Tailscale IP:   │
│ 100.x.x.10      │     │ 100.x.x.20      │     │ 100.x.x.30      │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────────────┴───────────────────────┘
                        Tailscale Mesh Network
                     (fonctionne partout dans le monde)
```

### 2.2 Créer un compte Tailscale

1. Va sur [https://tailscale.com](https://tailscale.com)
2. Crée un compte (GitHub, Google, ou email)
3. Note ton "Tailnet name" (ex: `joey-pilot.ts.net`)

### 2.3 Installation sur le VPS (Ubuntu 24.04)

```bash
# Ajouter le dépôt Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Démarrer Tailscale
sudo tailscale up

# Tu verras un lien d'authentification - ouvre-le dans ton navigateur
# Exemple: https://login.tailscale.com/a/xxxxxxxxxxxx
```

Après authentification :

```bash
# Vérifier l'IP Tailscale du VPS
tailscale ip -4
# Exemple output: 100.100.100.30

# Vérifier le statut
tailscale status
```

### 2.4 Installation sur le Steam Deck (ArchLinux)

En mode Desktop sur ton Steam Deck :

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

# Se connecter
sudo tailscale up

# Réactiver le read-only (optionnel mais recommandé)
sudo steamos-readonly enable
```

> **Alternative sans désactiver read-only** : Utiliser Flatpak ou l'app Tailscale depuis Discover.

### 2.5 Installation sur Windows (PC Laragon)

1. Télécharge l'installer depuis [tailscale.com/download](https://tailscale.com/download)
2. Installe et connecte-toi avec le même compte
3. L'icône Tailscale apparaît dans la barre des tâches

### 2.6 Configuration Tailscale pour le VPS

Quelques options utiles sur le VPS :

```bash
# Accepter les routes (si tu veux accéder à d'autres machines via le VPS)
sudo tailscale up --accept-routes

# Désactiver l'expiration de la clé (important pour un serveur)
# Va sur https://login.tailscale.com/admin/machines
# Clique sur ton VPS → "Disable key expiry"
```

### 2.7 Tester la connectivité

Depuis ton Steam Deck ou PC :

```bash
# Ping le VPS via Tailscale
ping 100.x.x.30  # Remplace par l'IP Tailscale de ton VPS

# SSH via Tailscale (fonctionne de n'importe où dans le monde!)
ssh -p 2222 root@100.x.x.30
```

---

## 3. Gestion des clés SSH multi-devices

### 3.1 Pourquoi une clé par device ?

- **Révocation indépendante** : Si tu perds ton Steam Deck, tu révoques uniquement sa clé
- **Traçabilité** : Tu sais quel device s'est connecté
- **Sécurité** : Pas de copie de clé privée entre machines

### 3.2 Générer la clé sur le PC Windows (Laragon)

```powershell
# PowerShell ou Git Bash
ssh-keygen -t ed25519 -C "joey-pc-windows" -f ~/.ssh/id_ed25519_vps
```

Cela crée :
- `~/.ssh/id_ed25519_vps` (clé privée)
- `~/.ssh/id_ed25519_vps.pub` (clé publique)

### 3.3 Générer la clé sur le Steam Deck (ArchLinux)

```bash
# En mode Desktop, ouvre Konsole
ssh-keygen -t ed25519 -C "joey-steamdeck" -f ~/.ssh/id_ed25519_vps

# Afficher la clé publique pour la copier
cat ~/.ssh/id_ed25519_vps.pub
```

### 3.4 Configurer le fichier SSH config (sur chaque device)

Crée/édite `~/.ssh/config` :

**Sur Windows** (`C:\Users\Joey\.ssh\config`) :

```
Host vps
    HostName 100.x.x.30
    Port 2222
    User root
    IdentityFile ~/.ssh/id_ed25519_vps

Host vps-public
    HostName ton-ip-publique-vps
    Port 2222
    User root
    IdentityFile ~/.ssh/id_ed25519_vps
```

**Sur Steam Deck** (`~/.ssh/config`) :

```
Host vps
    HostName 100.x.x.30
    Port 2222
    User root
    IdentityFile ~/.ssh/id_ed25519_vps

Host vps-public
    HostName ton-ip-publique-vps
    Port 2222
    User root
    IdentityFile ~/.ssh/id_ed25519_vps
```

Maintenant tu peux simplement faire :

```bash
ssh vps
# Au lieu de : ssh -p 2222 -i ~/.ssh/id_ed25519_vps root@100.x.x.30
```

### 3.5 Ajouter les clés publiques sur le VPS

Sur le VPS, ajoute les deux clés publiques :

```bash
nano ~/.ssh/authorized_keys
```

Ajoute une ligne par clé :

```
ssh-ed25519 AAAA...reste-de-la-clé... joey-pc-windows
ssh-ed25519 AAAA...reste-de-la-clé... joey-steamdeck
```

### 3.6 Permissions correctes

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 3.7 Révocation d'urgence (si perte d'un device)

Si tu perds ton Steam Deck :

```bash
# Sur le VPS, édite authorized_keys
nano ~/.ssh/authorized_keys
# Supprime la ligne contenant "joey-steamdeck"
```

---

## 4. Hardening SSH

> Les clés SSH sont déjà gérées dans la section 3. Ici on configure le serveur SSH.

### 4.1 Configuration SSH sécurisée

```bash
sudo nano /etc/ssh/sshd_config
```

Modifier/ajouter ces lignes :

```bash
# Changer le port (choisis un port entre 10000-65535)
Port 2222

# Désactiver l'accès root par mot de passe
PermitRootLogin prohibit-password

# Désactiver l'authentification par mot de passe (APRÈS avoir testé tes clés!)
PasswordAuthentication no

# Autres paramètres de sécurité
MaxAuthTries 3
MaxSessions 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Désactiver les méthodes d'auth non utilisées
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no

# Autoriser uniquement certains utilisateurs (optionnel)
# AllowUsers root ton-user
```

### 4.2 Tester la configuration SSH

```bash
sudo sshd -t
```

Si pas d'erreur :

```bash
sudo systemctl restart sshd
```

> ⚠️ **IMPORTANT** : Garde ta session SSH actuelle ouverte et ouvre une NOUVELLE session pour tester la connexion sur le nouveau port avant de fermer l'ancienne !

```bash
# Via Tailscale (recommandé - fonctionne de n'importe où)
ssh vps

# Ou directement
ssh -p 2222 -i ~/.ssh/id_ed25519_vps root@100.x.x.30
```

---

## 5. Configuration UFW via CloudPanel

### 5.1 Comprendre la gestion UFW de CloudPanel

> ⚠️ **IMPORTANT** : CloudPanel est le "master" du firewall UFW. Il stocke ses règles dans sa base SQLite (`/home/clp/htdocs/app/data/db.sq3`) et peut **écraser** les règles ajoutées directement en CLI !

**Règle d'or** : Utilise l'interface CloudPanel pour toutes les règles standards. Réserve la CLI uniquement pour les règles avancées (interface Tailscale).

### 5.2 Accéder à l'interface Firewall CloudPanel

1. Connecte-toi à CloudPanel : `https://ton-ip:8443`
2. Va dans **Admin Area** → **Security** → **Firewall**
3. Tu verras les règles existantes (22, 80, 443, 8443 par défaut)

### 5.3 Règles à configurer via l'interface CloudPanel

Clique sur **Add Rule** pour chaque règle :

| Port Range | Source | Description |
|------------|--------|-------------|
| `2222` | `0.0.0.0/0` | SSH Custom Port |
| `80` | `0.0.0.0/0` | HTTP |
| `443` | `0.0.0.0/0` | HTTPS |
| `8443` | `100.64.0.0/10` | CloudPanel (Tailscale only) |

> 💡 **Astuce sécurité** : En limitant 8443 à `100.64.0.0/10`, seules les connexions via Tailscale peuvent accéder à CloudPanel. Plus besoin de Basic Auth en front !

### 5.4 Supprimer les anciennes règles trop permissives

Dans l'interface CloudPanel Firewall, supprime les règles par défaut trop ouvertes :
- Supprime la règle `8443` avec source `0.0.0.0/0` (remplacée par notre règle Tailscale)
- Garde `22` ouvert pour le moment (on le sécurisera après avoir vérifié que Tailscale fonctionne)

### 5.5 Règle Tailscale (via CLI - nécessaire)

L'interface CloudPanel ne permet pas de créer des règles sur une **interface réseau** (`tailscale0`). Cette règle doit être ajoutée en CLI :

```bash
# Autoriser TOUT le trafic entrant depuis l'interface Tailscale
sudo ufw allow in on tailscale0 comment 'Tailscale Interface'

# Vérifier
sudo ufw status | grep tailscale
```

> ⚠️ Cette règle peut être perdue si CloudPanel modifie UFW. On va la protéger.

### 5.6 Protéger les règles CLI avec un script de restauration

Crée un script qui sera exécuté régulièrement pour s'assurer que la règle Tailscale existe :

```bash
sudo nano /usr/local/bin/ensure-tailscale-ufw.sh
```

```bash
#!/bin/bash
# Ensure Tailscale UFW rule exists
# This rule cannot be added via CloudPanel GUI

if ! ufw status | grep -q "tailscale0"; then
    ufw allow in on tailscale0 comment 'Tailscale Interface'
    echo "$(date): Tailscale UFW rule restored" >> /var/log/tailscale-ufw.log
fi
```

```bash
sudo chmod +x /usr/local/bin/ensure-tailscale-ufw.sh
```

Ajoute-le au cron de root :

```bash
sudo crontab -e
```

Ajoute ces lignes :

```bash
# Vérifier la règle Tailscale au reboot et toutes les heures
@reboot /usr/local/bin/ensure-tailscale-ufw.sh
0 * * * * /usr/local/bin/ensure-tailscale-ufw.sh
```

### 5.7 Configuration finale recommandée

Après configuration, `sudo ufw status` devrait montrer :

```
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere                   # SSH Custom
80/tcp                     ALLOW       Anywhere                   # HTTP
443/tcp                    ALLOW       Anywhere                   # HTTPS
8443/tcp                   ALLOW       100.64.0.0/10              # CloudPanel Tailscale
Anywhere on tailscale0     ALLOW       Anywhere                   # Tailscale Interface
```

### 5.8 Sécuriser SSH après validation Tailscale

Une fois que tu as confirmé que Tailscale fonctionne parfaitement :

1. Dans CloudPanel Firewall, **supprime** la règle `22` source `0.0.0.0/0`
2. Le SSH restera accessible via Tailscale grâce à la règle `tailscale0`
3. Ton port custom `2222` reste accessible publiquement (avec protection CrowdSec)

### 5.9 Accès d'urgence (si Tailscale échoue)

**Option 1** : Console VNC/Serial Hostinger
- Va sur hPanel Hostinger → VPS → Console
- Tu as un accès direct sans passer par le réseau

**Option 2** : Règle temporaire via CloudPanel
- Si tu as accès à une IP publique stable, ajoute-la temporairement
- Supprime-la après usage

---

## 6. Installation CrowdSec

### 6.1 Ajout du dépôt officiel CrowdSec

> ⚠️ **Important** : Ubuntu 24.04 a une version obsolète (1.4.6) dans ses dépôts. Nous utilisons le dépôt officiel pour avoir la dernière version.

```bash
curl -s https://install.crowdsec.net | sudo sh
```

### 6.2 Configuration du pinning APT (priorité au dépôt CrowdSec)

```bash
sudo nano /etc/apt/preferences.d/crowdsec
```

Ajouter :

```
Package: *
Pin: release o=packagecloud.io/crowdsec/crowdsec,a=any,n=any,c=main
Pin-Priority: 1001
```

### 6.3 Mise à jour et installation

```bash
sudo apt update

# Vérifier que la bonne version sera installée
apt-cache policy crowdsec

# Installer CrowdSec
sudo apt install crowdsec -y
```

### 6.4 Vérification de l'installation

```bash
sudo systemctl status crowdsec
sudo cscli version
```

---

## 7. Configuration pour CloudPanel

### 7.1 Structure des logs CloudPanel

CloudPanel organise les logs ainsi :
```
/home/{site-user}/logs/
├── nginx/
│   ├── access.log
│   └── error.log
└── php/
    └── error.log
```

Plus les logs globaux :
```
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/auth.log
/var/log/syslog
```

### 7.2 Configuration de l'acquisition des logs

```bash
sudo nano /etc/crowdsec/acquis.yaml
```

Remplacer le contenu par :

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
# Uses glob pattern to capture all site logs
#-----------------------------------------
filenames:
  - /home/*/logs/nginx/access.log
  - /home/*/logs/nginx/error.log
labels:
  type: nginx
---
#-----------------------------------------
# CloudPanel - All sites PHP logs (optionnel)
#-----------------------------------------
filenames:
  - /home/*/logs/php/error.log
labels:
  type: syslog
---
#-----------------------------------------
# Journald for systemd services
#-----------------------------------------
journalctl_filter:
  - "_SYSTEMD_UNIT=ssh.service"
  - "_SYSTEMD_UNIT=sshd.service"
labels:
  type: syslog
```

### 7.3 Vérification des permissions

CrowdSec doit pouvoir lire les logs. Vérifie :

```bash
# Tester l'accès aux logs
sudo -u crowdsec cat /var/log/nginx/access.log > /dev/null && echo "OK" || echo "ERREUR"

# Si erreur, ajouter crowdsec au groupe adm
sudo usermod -aG adm crowdsec
```

### 7.4 Redémarrage de CrowdSec

```bash
sudo systemctl restart crowdsec
sudo systemctl status crowdsec
```

---

## 8. Installation des Bouncers

### 8.1 Firewall Bouncer (nftables - recommandé pour Ubuntu 24.04)

```bash
sudo apt install crowdsec-firewall-bouncer-nftables -y
sudo systemctl enable crowdsec-firewall-bouncer
sudo systemctl start crowdsec-firewall-bouncer
```

Vérification :

```bash
sudo systemctl status crowdsec-firewall-bouncer
sudo cscli bouncers list
```

### 8.2 Nginx Bouncer (optionnel - niveau applicatif)

Le bouncer Nginx permet de bloquer au niveau HTTP et d'afficher des captchas. C'est une couche supplémentaire au firewall bouncer.

```bash
sudo apt install crowdsec-nginx-bouncer -y
```

Configuration :

```bash
sudo nano /etc/crowdsec/bouncers/crowdsec-nginx-bouncer.conf
```

Vérifier que les paramètres sont corrects :

```yaml
api_url: http://127.0.0.1:8080/
api_key: <auto-generated>
# Mode: ban, captcha, or allow
mode: live
# Enable recaptcha (optionnel)
# recaptcha_enabled: true
# recaptcha_site_key: <your-key>
# recaptcha_secret_key: <your-secret>
```

Redémarrer Nginx :

```bash
sudo systemctl restart nginx
sudo cscli bouncers list
```

> **Note** : Le bouncer Nginx modifie la config Nginx. Si tu as des problèmes, vérifie `/etc/nginx/conf.d/crowdsec_nginx.conf`.

---

## 9. Collections et Scénarios

### 9.1 Installation des collections essentielles

```bash
# Collection Linux (SSH, système)
sudo cscli collections install crowdsecurity/linux

# Collection Nginx
sudo cscli collections install crowdsecurity/nginx

# Collection HTTP générique (bots, scans, etc.)
sudo cscli collections install crowdsecurity/http-cve

# Collection base-http-scenarios (exploits web communs)
sudo cscli collections install crowdsecurity/base-http-scenarios

# WhiteLists (pour éviter les faux positifs)
sudo cscli parsers install crowdsecurity/whitelists
```

### 9.2 Scénarios additionnels recommandés

```bash
# Brute force SSH
sudo cscli scenarios install crowdsecurity/ssh-bf
sudo cscli scenarios install crowdsecurity/ssh-slow-bf

# Probing HTTP
sudo cscli scenarios install crowdsecurity/http-probing
sudo cscli scenarios install crowdsecurity/http-sensitive-files
sudo cscli scenarios install crowdsecurity/http-bad-user-agent

# CVE exploits
sudo cscli scenarios install crowdsecurity/http-cve-2021-41773
sudo cscli scenarios install crowdsecurity/http-cve-2021-42013

# Enumération
sudo cscli scenarios install crowdsecurity/http-path-traversal-probing
```

### 9.3 Voir ce qui est installé

```bash
sudo cscli collections list
sudo cscli scenarios list
sudo cscli parsers list
```

### 9.4 Recharger CrowdSec

```bash
sudo systemctl reload crowdsec
```

---

## 10. CrowdSec Console

La Console CrowdSec est un dashboard web gratuit pour monitorer tes instances.

### 10.1 Créer un compte

1. Va sur [https://app.crowdsec.net](https://app.crowdsec.net)
2. Crée un compte gratuit
3. Dans "Security Engines", clique sur "Add Security Engine"
4. Copie la clé d'enrollment

### 10.2 Enroller ton serveur

```bash
sudo cscli console enroll <TA-CLE-DENROLLMENT>
```

Exemple :

```bash
sudo cscli console enroll cl7xxxxxxxxxxxxxxxxxxxxx
```

### 10.3 Valider l'enrollment

Retourne sur la Console web et accepte la demande d'enrollment.

### 10.4 Activer le partage de données (recommandé)

```bash
sudo cscli console enable -a
sudo systemctl reload crowdsec
```

Cela active :
- Le partage d'alertes avec la communauté
- La réception de la blocklist communautaire

### 10.5 Vérifier le statut

```bash
sudo cscli console status
```

---

## 11. Whitelisting intelligent

### 11.1 Whitelister le réseau Tailscale (CRUCIAL)

C'est ici que la magie opère pour ton workflow nomade. En whitelistant le subnet Tailscale, tu ne seras **jamais** bloqué, peu importe depuis quel WiFi d'hôtel tu te connectes.

```bash
sudo nano /etc/crowdsec/parsers/s02-enrich/my-whitelists.yaml
```

```yaml
name: crowdsecurity/my-whitelists
description: "Custom whitelists for nomad developer"
whitelist:
  reason: "Tailscale network - always trusted"
  cidr:
    # Tailscale CGNAT range - TOUTES tes connexions Tailscale
    - "100.64.0.0/10"
    
    # Réseaux privés standards (au cas où)
    - "10.0.0.0/8"
    - "172.16.0.0/12"
    - "192.168.0.0/16"
    
    # Localhost
    - "127.0.0.0/8"
```

> **Pourquoi `100.64.0.0/10` ?** C'est le range CGNAT utilisé par Tailscale pour toutes les IPs de ton réseau mesh. Que tu sois à Tokyo, New York ou Paris, ton Steam Deck aura toujours une IP dans ce range.

### 11.2 Whitelist additionnelle (optionnel)

Si tu as une IP fixe à domicile :

```bash
sudo nano /etc/crowdsec/parsers/s02-enrich/home-whitelist.yaml
```

```yaml
name: crowdsecurity/home-whitelist
description: "Home IP whitelist"
whitelist:
  reason: "Home static IP"
  ip:
    - "1.2.3.4"  # Ton IP fixe domicile (si tu en as une)
```

### 11.3 Appliquer les whitelists

```bash
sudo systemctl reload crowdsec

# Vérifier que les parsers sont chargés
sudo cscli parsers list | grep whitelist
```

### 11.4 Whitelist express (si tu te fais bannir accidentellement)

```bash
# Voir les décisions actives
sudo cscli decisions list

# Supprimer un ban spécifique
sudo cscli decisions delete -i 100.x.x.x

# Si tu ne peux plus accéder au VPS :
# 1. Utilise la console Hostinger (VNC/Serial Console)
# 2. Ou connecte-toi via l'IP publique depuis un autre réseau
```

---

## 12. Mises à jour automatiques

### 12.1 Mises à jour de sécurité Ubuntu

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

Choisir "Yes" pour activer.

### 12.2 Configuration avancée (optionnel)

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Ajouter les dépôts CrowdSec si souhaité :

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
    "packagecloud.io/crowdsec/crowdsec:any";
};
```

### 12.3 Mise à jour automatique du hub CrowdSec

CrowdSec peut mettre à jour ses scénarios automatiquement :

```bash
# Créer un cron job
sudo crontab -e
```

Ajouter :

```bash
# Mise à jour hub CrowdSec tous les jours à 4h du matin
0 4 * * * /usr/bin/cscli hub update && /usr/bin/cscli hub upgrade --all
```

---

## 13. Commandes de maintenance

### 13.1 Monitoring quotidien

```bash
# Voir les métriques globales
sudo cscli metrics

# Alertes des dernières 24h
sudo cscli alerts list --since 24h

# Décisions actives (IPs bannies)
sudo cscli decisions list

# Statut des services
sudo systemctl status crowdsec
sudo systemctl status crowdsec-firewall-bouncer
```

### 13.2 Gestion des décisions

```bash
# Bannir une IP manuellement (4 heures)
sudo cscli decisions add -i 1.2.3.4 -t ban -d 4h -r "Manual ban: suspicious activity"

# Bannir un range IP
sudo cscli decisions add -r 1.2.3.0/24 -t ban -d 24h -r "Range ban"

# Supprimer un ban
sudo cscli decisions delete -i 1.2.3.4

# Supprimer tous les bans
sudo cscli decisions delete --all
```

### 13.3 Inspection des alertes

```bash
# Liste des alertes
sudo cscli alerts list

# Détail d'une alerte
sudo cscli alerts inspect <ALERT_ID>

# Supprimer les anciennes alertes
sudo cscli alerts delete --all
```

### 13.4 Mise à jour du hub

```bash
# Mettre à jour l'index
sudo cscli hub update

# Lister les mises à jour disponibles
sudo cscli hub list -a

# Tout mettre à jour
sudo cscli hub upgrade --all

# Recharger après mise à jour
sudo systemctl reload crowdsec
```

### 13.5 Debug

```bash
# Logs CrowdSec
sudo journalctl -u crowdsec -e --no-pager

# Logs du bouncer firewall
sudo journalctl -u crowdsec-firewall-bouncer -e --no-pager

# Tester le parsing des logs
sudo cscli explain --file /var/log/nginx/access.log --type nginx
```

---

## 14. Tests et validation

### 14.1 Test du firewall bouncer

Depuis une autre machine (ou via un VPN) :

```bash
# Tente plusieurs connexions SSH échouées
for i in {1..10}; do ssh -p 2222 fakeuser@ton-ip-vps; done
```

Puis sur ton serveur :

```bash
sudo cscli decisions list
# Tu devrais voir l'IP bannie
```

### 14.2 Test du scénario HTTP

```bash
# Simulation d'un scan Nikto (depuis une autre machine)
curl -A "Nikto" http://ton-domaine.com/
curl http://ton-domaine.com/wp-admin/
curl http://ton-domaine.com/../../../etc/passwd
```

Vérifier :

```bash
sudo cscli alerts list --since 1h
```

### 14.3 Test Tailscale (IMPORTANT)

Depuis ton Steam Deck connecté à un WiFi quelconque :

```bash
# Vérifie que Tailscale est connecté
tailscale status

# Test SSH via Tailscale
ssh vps

# Tu devrais te connecter instantanément, sans être bloqué par CrowdSec
```

### 14.4 Vérification que tout fonctionne

```bash
# Métriques - vérifie que les logs sont parsés
sudo cscli metrics

# Tu devrais voir des lignes parsed pour nginx et syslog
```

Output attendu (exemple) :

```
╭─────────────────────────────────────────────────────────────╮
│                        Acquisition Metrics                   │
├─────────────────────────────────────────────────────────────┤
│ Source                                │ Lines read │ Lines parsed │
├───────────────────────────────────────┼────────────┼──────────────┤
│ file:/var/log/nginx/access.log        │ 1234       │ 1234         │
│ file:/var/log/auth.log                │ 567        │ 567          │
│ file:/home/*/logs/nginx/access.log    │ 890        │ 890          │
╰───────────────────────────────────────┴────────────┴──────────────╯
```

---

## 15. Workflow nomade - Checklist escale

### Ta routine à chaque nouvelle destination

**Temps total : ~30 secondes** ⏱️

```
┌─────────────────────────────────────────────────────────────────┐
│  🛬 ARRIVÉE À L'HÔTEL                                          │
├─────────────────────────────────────────────────────────────────┤
│  1. Connecte-toi au WiFi de l'hôtel                            │
│  2. Ouvre le Steam Deck en mode Desktop                        │
│  3. Vérifie l'icône Tailscale (barre des tâches)               │
│     → Si déconnecté : clic droit → Connect                     │
│  4. Ouvre Konsole et tape : ssh vps                            │
│  5. Tu es connecté ! 🎉                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Pourquoi ça "just works"

| Problème classique | Solution Tailscale |
|-------------------|-------------------|
| IP d'hôtel inconnue | Tailscale traverse le NAT automatiquement |
| Firewall d'hôtel restrictif | Tailscale utilise DERP relay si nécessaire |
| CrowdSec pourrait bloquer | Subnet `100.64.0.0/10` whitelisté |
| Clé SSH différente par device | SSH config avec `Host vps` |

### En cas de problème

```bash
# 1. Tailscale ne se connecte pas ?
tailscale status
sudo systemctl restart tailscaled

# 2. SSH timeout ?
# Vérifie que le VPS est dans ton réseau Tailscale
tailscale ping 100.x.x.30  # IP Tailscale du VPS

# 3. "Connection refused" ?
# Le VPS est peut-être reboot. Attends 2-3 minutes.

# 4. Tu t'es fait bannir ? (peu probable avec Tailscale)
# Utilise la console VNC de Hostinger pour débannir
sudo cscli decisions delete --all
```

### Backup : Accès via IP publique

Si Tailscale a un souci (rare), tu peux toujours utiliser l'IP publique :

```bash
ssh vps-public
```

> ⚠️ Attention : Sans Tailscale, ton IP d'hôtel n'est pas whitelistée. Évite les erreurs de mot de passe !

---

## Récapitulatif de l'architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         INTERNET                                │
│                                                                 │
│    ┌──────────────┐         ┌──────────────┐                   │
│    │ Steam Deck   │         │  PC Windows  │                   │
│    │ (Tokyo)      │         │  (Paris)     │                   │
│    │ WiFi Hôtel   │         │  Domicile    │                   │
│    └──────┬───────┘         └──────┬───────┘                   │
│           │                        │                           │
│           └────────┬───────────────┘                           │
│                    │                                           │
│           ┌────────▼────────┐                                  │
│           │   TAILSCALE     │                                  │
│           │   Mesh Network  │                                  │
│           │  100.64.0.0/10  │                                  │
│           └────────┬────────┘                                  │
│                    │ (chiffré WireGuard)                       │
└────────────────────┼───────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VPS HOSTINGER                                │
├─────────────────────────────────────────────────────────────────┤
│  UFW (géré via CloudPanel GUI + script Tailscale)              │
│      → 80, 443, 2222 : ouverts au public                       │
│      → 8443 : Tailscale uniquement (100.64.0.0/10)             │
│      → tailscale0 : tout autorisé (via script cron)            │
├─────────────────────────────────────────────────────────────────┤
│  CrowdSec Firewall Bouncer                                      │
│      → 100.64.0.0/10 WHITELISTÉ (jamais banni via Tailscale)   │
│      → Autres IPs : analyse et ban si malveillantes            │
├─────────────────────────────────────────────────────────────────┤
│  NGINX (CloudPanel)                                             │
│      → Windshear Ahead, Piwigo, Laravel apps                   │
├─────────────────────────────────────────────────────────────────┤
│  CrowdSec Engine                                                │
│      → Analyse logs, partage CTI                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Checklist finale

### Installation serveur
- [ ] SSH sur port personnalisé (2222)
- [ ] Authentification SSH par clé uniquement
- [ ] UFW configuré via **interface CloudPanel** (Admin → Security → Firewall)
- [ ] Règle 8443 limitée à `100.64.0.0/10` (Tailscale only)
- [ ] Script `/usr/local/bin/ensure-tailscale-ufw.sh` créé et dans cron
- [ ] CrowdSec installé depuis le dépôt officiel
- [ ] acquis.yaml configuré pour CloudPanel (`/home/*/logs/nginx/*.log`)
- [ ] Firewall bouncer (nftables) installé et actif
- [ ] Collections linux et nginx installées
- [ ] Console CrowdSec connectée
- [ ] Whitelist Tailscale `100.64.0.0/10` dans CrowdSec
- [ ] Mises à jour automatiques configurées

### Configuration Tailscale
- [ ] Compte Tailscale créé
- [ ] Tailscale installé sur VPS
- [ ] Tailscale installé sur PC Windows
- [ ] Tailscale installé sur Steam Deck
- [ ] "Disable key expiry" activé pour le VPS
- [ ] Test de connexion SSH via Tailscale réussi

### Clés SSH
- [ ] Clé SSH générée sur PC Windows (`joey-pc-windows`)
- [ ] Clé SSH générée sur Steam Deck (`joey-steamdeck`)
- [ ] Les deux clés ajoutées dans `authorized_keys` du VPS
- [ ] Fichier `~/.ssh/config` configuré sur les deux devices

### Tests finaux
- [ ] `ssh vps` fonctionne depuis PC Windows
- [ ] `ssh vps` fonctionne depuis Steam Deck (via WiFi quelconque)
- [ ] Test de ban SSH réussi (depuis IP non-Tailscale)
- [ ] Métriques CrowdSec affichent des logs parsés

---

## Ressources

- [Documentation CrowdSec](https://docs.crowdsec.net/)
- [Hub CrowdSec (scénarios)](https://hub.crowdsec.net/)
- [Console CrowdSec](https://app.crowdsec.net/)
- [Documentation CloudPanel](https://www.cloudpanel.io/docs/v2/)
- [Documentation Tailscale](https://tailscale.com/kb/)
- [Tailscale Admin Console](https://login.tailscale.com/admin/)

---

*Guide créé le 24/12/2024 - Adapté pour Ubuntu 24.04 LTS + CloudPanel + Tailscale (workflow nomade)*
