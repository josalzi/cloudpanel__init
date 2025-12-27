# Ajouter un site Laravel sur CloudPanel avec GitHub

## 1. Créer le site dans CloudPanel

1. **Sites → Add Site → Create a PHP Site**
2. Configuration :
   - **Domain Name** : `mon-site.domaine.com`
   - **PHP Version** : 8.3 (ou 8.4/8.5 selon ton projet)
   - **Document Root** : `/public`
   - Cocher **Create Database** si nécessaire
3. Valider

CloudPanel crée automatiquement :
- Un utilisateur système (ex: `mon-site`)
- Un répertoire `/home/mon-site/htdocs/mon-site.domaine.com/`

---

## 2. Générer la clé SSH pour GitHub

```bash
# Se connecter en tant que site-user
su - mon-site

# Générer la clé ED25519
ssh-keygen -t ed25519 -C "deploy-mon-site"
```

Appuie sur **Entrée** pour accepter le chemin par défaut, puis **Entrée** deux fois (pas de passphrase).

```bash
# Afficher la clé publique
cat ~/.ssh/id_ed25519.pub
```

**Copie toute la ligne** affichée (de `ssh-ed25519` jusqu'à `deploy-mon-site`).

---

## 3. Ajouter la clé comme Deploy Key sur GitHub

1. Va sur ton repo GitHub → **Settings → Deploy keys → Add deploy key**
2. **Title** : `CloudPanel - mon-site` (ou ce que tu veux)
3. **Key** : Colle la clé publique copiée
4. Coche **Allow write access** si tu veux pouvoir push depuis le serveur
5. **Add key**

---

## 4. Cloner le projet

```bash
# Toujours en tant que site-user
cd ~/htdocs/mon-site.domaine.com

# Supprimer les fichiers par défaut de CloudPanel
rm -rf *

# Cloner le repo
git clone git@github.com:ton-username/ton-repo.git .
```

> **Note** : Le `.` à la fin clone directement dans le dossier courant.

Si c'est la première connexion SSH à GitHub :
```
Are you sure you want to continue connecting (yes/no)?
```
Réponds `yes`.

---

## 5. Installer les dépendances

```bash
# Composer
composer install --no-interaction --prefer-dist --optimize-autoloader --no-dev

# NPM
npm install
npm run build
```

---

## 6. Configurer Laravel

```bash
# Copier le fichier d'environnement
cp .env.example .env

# Éditer la configuration
nano .env
```

**Variables essentielles à modifier :**
```env
APP_NAME="Mon Site"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://mon-site.domaine.com

DB_DATABASE=xxx        # depuis CloudPanel → Databases
DB_USERNAME=xxx        # depuis CloudPanel → Databases
DB_PASSWORD=xxx        # depuis CloudPanel → Databases
```

```bash
# Générer la clé d'application
php artisan key:generate

# Lancer les migrations
php artisan migrate --force

# Deployer en utilisant le script Bash
bash atomic-deploy.sh
```

---

## 7. Permissions

```bash
chmod -R 755 storage bootstrap/cache
```

---

## Vhost CloudPanel

```bash
# ===========================================
# REDIRECT: lambda-aero.com → www.
# ===========================================
server {
  listen 80;
  listen [::]:80;
  listen 443 quic;
  listen 443 ssl;
  listen [::]:443 quic;
  listen [::]:443 ssl;
  http2 on;
  http3 off;
  {{ssl_certificate_key}}
  {{ssl_certificate}}
  server_name lambda-aero.com;
  
  return 301 https://www.lambda-aero.com$request_uri;
}

# ===========================================
# MAIN: www.codebase.lambda-aero.com
# ===========================================
server {
  listen 80;
  listen [::]:80;
  listen 443 quic;
  listen 443 ssl;
  listen [::]:443 quic;
  listen [::]:443 ssl;
  http2 on;
  http3 off;
  {{ssl_certificate_key}}
  {{ssl_certificate}}
  server_name www.lambda-aero.com;
  {{root}}

  {{nginx_access_log}}
  {{nginx_error_log}}

  if ($scheme != "https") {
    rewrite ^ https://$host$request_uri permanent;
  }

  location ~ /.well-known {
    auth_basic off;
    allow all;
  }

  {{settings}}

  location / {
    {{varnish_proxy_pass}}
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_hide_header X-Varnish;
    proxy_redirect off;
    proxy_max_temp_file_size 0;
    proxy_connect_timeout      720;
    proxy_send_timeout         720;
    proxy_read_timeout         720;
    proxy_buffer_size          128k;
    proxy_buffers              4 256k;
    proxy_busy_buffers_size    256k;
    proxy_temp_file_write_size 256k;
  }

  location ~* ^.+\.(css|js|jpg|jpeg|gif|png|ico|gz|svg|svgz|ttf|otf|woff|woff2|eot|mp4|ogg|ogv|webm|webp|zip|swf|map|mjs)$ {
    add_header Access-Control-Allow-Origin "*";
    add_header alt-svc 'h3=":443"; ma=86400';
    expires max;
    access_log off;
  }

  location ~ /\.(ht|svn|git) {
    deny all;
  }

  if (-f $request_filename) {
    break;
  }
}

# ===========================================
# BACKEND: PHP-FPM (Varnish upstream)
# ===========================================
server {
  listen 8080;
  listen [::]:8080;
  server_name www.lambda-aero.com;
  {{root}}

  include /etc/nginx/global_settings;

  try_files $uri $uri/ /index.php?$args;
  index index.php index.html;

  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_intercept_errors on;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    try_files $uri =404;
    fastcgi_read_timeout 3600;
    fastcgi_send_timeout 3600;
    fastcgi_param HTTPS "on";
    fastcgi_param SERVER_PORT 443;
    fastcgi_pass 127.0.0.1:{{php_fpm_port}};
    fastcgi_param PHP_VALUE "{{php_settings}}";
  }

  if (-f $request_filename) {
    break;
  }
}
```
