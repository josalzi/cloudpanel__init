# CloudPanel — Installation on VPS KVM8 (Ubuntu 24.04)

This document covers the full initialization and installation of **CloudPanel** with **MariaDB** on a fresh Hostinger VPS KVM8 running **Ubuntu 24.04 LTS**.

---

## 1. Pre-requisites

### 1.1 — Fresh OS Install

CloudPanel **must** be installed on a clean Ubuntu 24.04 instance. It will not install correctly on a system that already has Nginx, Apache, MySQL, or MariaDB pre-installed.

From the Hostinger VPS dashboard:

1. Select the KVM8 VPS.
2. Go to **OS & Panel** → **Operating System**.
3. Choose **Ubuntu 24.04 64bit** (plain, without any panel).
4. Confirm the reinstall — this wipes the disk.

### 1.2 — DNS Preparation

Before starting, point your server hostname and any initial domains to the VPS IP address:

```
A    vps.example.com    → <VPS_IP>
A    panel.example.com  → <VPS_IP>
```

### 1.3 — Initial SSH Access

```bash
ssh root@<VPS_IP>
```

> At this stage, SSH runs on port **22** with root password authentication. This will be hardened later.

---

## 2. System Preparation

### 2.1 — Update the System

```bash
apt update && apt upgrade -y
```

### 2.2 — Set the Hostname

```bash
hostnamectl set-hostname vps.example.com
```

Edit `/etc/hosts` accordingly:

```bash
nano /etc/hosts
```

```
127.0.0.1   localhost
<VPS_IP>    vps.example.com
```

### 2.3 — Timezone Configuration

```bash
timedatectl set-timezone UTC
```

> UTC is the safest choice for a server, especially when dealing with Laravel's `APP_TIMEZONE` and database timestamps. Adjust if needed.

### 2.4 — Reboot

```bash
reboot
```

---

## 3. CloudPanel Installation

### 3.1 — Install CloudPanel with MariaDB

CloudPanel provides a single curl-based installer. The MariaDB variant:

```bash
curl -sS https://installer.cloudpanel.io/ce/v2/install.sh -o install.sh

echo "a]Rsym÷fE4[EiHp" | sudo bash install.sh --db-engine mariadb
```

> **Important:** always verify the current installer URL and checksum on [cloudpanel.io/docs](https://www.cloudpanel.io/docs/v2/getting-started/). The installer script and its hash **change between versions**. The hash shown above is an example — **do not blindly copy it**.

The installation takes approximately 5 to 10 minutes. It installs:

- **Nginx** (as reverse proxy and web server)
- **MariaDB 11.4**
- **PHP-FPM** (multiple versions: 7.1 through 8.4)
- **Varnish Cache**
- **Redis**
- **Node.js** (via nvm)
- **Certbot** (Let's Encrypt)
- **Postfix** (local MTA)

### 3.2 — Access the Panel

Once the installer finishes, CloudPanel is accessible on port **8443**:

```
https://<VPS_IP>:8443
```

> The browser will warn about a self-signed certificate — this is expected on first access.

### 3.3 — Create the Admin User

On the first visit, CloudPanel prompts for the creation of an admin account. Fill in:

- **Name**
- **Email**
- **Password** (use a strong one, this is the panel superadmin)

### 3.4 — Panel SSL Certificate

Once logged in, navigate to **Admin Area** → **Settings** and set up a Let's Encrypt certificate for the panel itself, using a dedicated subdomain like `panel.example.com`.

After this, access the panel via:

```
https://panel.example.com:8443
```

---

## 4. Post-Installation Configuration

### 4.1 — MariaDB Verification

Verify that MariaDB is running and accessible:

```bash
systemctl status mariadb
mariadb --version
```

CloudPanel manages MariaDB users and databases through the UI. The root credentials are stored in:

```bash
cat /root/.my.cnf
```

### 4.2 — PHP Versions

CloudPanel ships with multiple PHP versions. Verify available versions:

```bash
ls /etc/php/
```

For Laravel 12, you'll want **PHP 8.3** or **8.4**. When creating a site in CloudPanel, select the appropriate PHP version.

### 4.3 — Composer

Composer is pre-installed by CloudPanel. Verify:

```bash
composer --version
```

If absent or outdated:

```bash
curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
```

### 4.4 — Node.js & npm

CloudPanel installs Node.js via **nvm** for the `clp` user. For system-wide availability:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
node -v && npm -v
```

### 4.5 — Git

```bash
apt install -y git
git --version
```

---

## 5. Nginx Specifics

### 5.1 — Vhost Structure

CloudPanel manages Nginx vhosts automatically. Each site gets its own config in:

```
/etc/nginx/sites-enabled/<domain>.conf
```

The root directive for a Laravel site points to `/home/<user>/htdocs/<domain>/public`.

### 5.2 — Custom Nginx Directives

For Laravel + Livewire, you may need to add custom directives. In CloudPanel, go to the site → **Vhost** tab and add inside the `server` block:

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

> CloudPanel's default PHP site template usually handles this, but verify it's present. Without it, Laravel routing will 404 on anything except `/`.

### 5.3 — Varnish Consideration

Varnish is enabled by default on port **6081**, sitting in front of Nginx. For Livewire applications, Varnish **will cause issues** with CSRF tokens and dynamic content. Two options:

1. **Disable Varnish globally** (if all sites are Laravel/Livewire):
   ```bash
   systemctl stop varnish
   systemctl disable varnish
   ```
   Then in CloudPanel, edit each site's vhost to listen on port **80** directly instead of proxying through Varnish.

2. **Bypass Varnish per-site** by adjusting the vhost config so Nginx listens on 80/443 directly without the Varnish backend.

> Varnish and Livewire don't mix well. For Livewire-heavy sites, disabling Varnish is the cleanest path.

---

## 6. MariaDB Tuning

The KVM8 plan provides **8 vCPU / 32 GB RAM / 400 GB NVMe** — a significant upgrade from KVM2. MariaDB can be tuned accordingly.

### 6.1 — Configuration File

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

### 6.2 — Recommended Adjustments for KVM8

```ini
[mysqld]
innodb_buffer_pool_size = 8G
innodb_log_file_size = 1G
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

max_connections = 200
tmp_table_size = 256M
max_heap_table_size = 256M
table_open_cache = 4000
thread_cache_size = 16

query_cache_type = 0
query_cache_size = 0

join_buffer_size = 4M
sort_buffer_size = 4M
read_rnd_buffer_size = 2M
```

> `innodb_buffer_pool_size` at **8G** is a solid starting point for 32 GB total RAM, leaving plenty of headroom for Nginx, PHP-FPM workers, Redis, and the OS. Adjust based on actual workload.

### 6.3 — Restart MariaDB

```bash
systemctl restart mariadb
```

Verify:

```bash
mariadb -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
```

---

## 7. File Permissions & Directory Structure

### 7.1 — CloudPanel Site User

Each site created in CloudPanel gets a dedicated Linux user. The home directory structure:

```
/home/<site-user>/
├── htdocs/
│   └── <domain>/          ← Laravel project root
│       ├── public/        ← Nginx document root
│       ├── storage/
│       ├── bootstrap/cache/
│       └── ...
├── logs/
│   ├── <domain>/
│   │   ├── access.log
│   │   └── error.log
├── tmp/
└── .ssh/                  ← For Git deploy keys
```

### 7.2 — Laravel Storage Permissions

After deploying a Laravel app:

```bash
cd /home/<site-user>/htdocs/<domain>

chown -R <site-user>:<site-user> storage bootstrap/cache
chmod -R 775 storage bootstrap/cache
```

---

## 8. Creating a Laravel Site

### 8.1 — Via CloudPanel UI

1. **Add Site** → Choose **PHP** site type.
2. Set the **domain name**.
3. Select **PHP 8.3** (or 8.4).
4. CloudPanel creates the user, vhost, database (optional), and directory structure.

### 8.2 — Database Creation

In CloudPanel → **Databases**:

1. **Add Database**.
2. Name it, create a dedicated user with a strong password.
3. Note the credentials for `.env`.

### 8.3 — SSL Certificate

In the site settings → **SSL/TLS** → **Actions** → **New Let's Encrypt Certificate**.

Select the domain and `www` subdomain if needed.

### 8.4 — Deploy Laravel

Switch to the site user and clone the repo:

```bash
su - <site-user>
cd htdocs/

# Remove the default directory CloudPanel created
rm -rf <domain>

# Clone your repository
git clone git@github.com:josalzi/<repo>.git <domain>
cd <domain>

# Install dependencies
composer install --no-dev --optimize-autoloader
npm install && npm run build

# Environment setup
cp .env.example .env
php artisan key:generate

# Edit .env with database credentials, APP_URL, etc.
nano .env

# Migrations
php artisan migrate --force

# Permissions
chmod -R 775 storage bootstrap/cache
```

---

## 9. Cron & Queue Workers

### 9.1 — Laravel Scheduler

Add the Laravel scheduler cron for the site user:

```bash
crontab -e -u <site-user>
```

```cron
* * * * * cd /home/<site-user>/htdocs/<domain> && php artisan schedule:run >> /dev/null 2>&1
```

### 9.2 — Queue Worker (Supervisor)

Install Supervisor:

```bash
apt install -y supervisor
```

Create a worker config:

```bash
nano /etc/supervisor/conf.d/<domain>-worker.conf
```

```ini
[program:domain-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/<site-user>/htdocs/<domain>/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=<site-user>
numprocs=2
redirect_stderr=true
stdout_logfile=/home/<site-user>/logs/worker.log
stopwaitsecs=3600
```

```bash
supervisorctl reread
supervisorctl update
supervisorctl start domain-worker:*
```

---

## 10. Redis Configuration

Redis is installed by CloudPanel. Verify and configure:

```bash
systemctl status redis-server
```

For Laravel, update `.env`:

```env
CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

Ensure `phpredis` or `predis` is available:

```bash
php8.3-redis  # Usually pre-installed with CloudPanel's PHP
```

If not:

```bash
apt install -y php8.3-redis
systemctl restart php8.3-fpm
```

---

## 11. Summary — Installation Checklist

| Step | Action | Status |
|------|--------|--------|
| 1 | Fresh Ubuntu 24.04 install | ☐ |
| 2 | System update + hostname + timezone | ☐ |
| 3 | CloudPanel install (MariaDB variant) | ☐ |
| 4 | Admin account creation | ☐ |
| 5 | Panel SSL certificate | ☐ |
| 6 | MariaDB tuning for KVM8 | ☐ |
| 7 | PHP version verification | ☐ |
| 8 | Composer / Node.js / Git | ☐ |
| 9 | Varnish assessment (disable for Livewire) | ☐ |
| 10 | First Laravel site creation | ☐ |
| 11 | Redis configuration | ☐ |
| 12 | Supervisor + cron setup | ☐ |

> **Next steps:** SSH hardening, Tailscale, CrowdSec, UFW firewall, and webhook auto-deployment — covered in separate documents.
