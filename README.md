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

# Optimiser pour la production
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Publier les assets Livewire (si utilisé)
php artisan livewire:publish --assets
```

---

## 7. Permissions

```bash
chmod -R 755 storage bootstrap/cache
```

---

## Commandes utiles

```bash
# Mettre à jour depuis GitHub
git pull origin main

# Réoptimiser après modification
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Vider les caches
php artisan cache:clear
php artisan config:clear
```
