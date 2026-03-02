# Python

## Required for tile generation
```bash
apt install python3-numpy python3-pillow python3-pyproj gdal-bin libgdal-dev python3-gdal python3-rasterio python3-scipy python3-psutil python3-pandas python3-h5py
```

## Optional (for BUFR inspection)
```bash
apt install python3-eccodes python3-cfgrib
```


# img2webp

```bash
sudo apt install webp
```

# Playwright

```bash
sudo apt-get install -y \
    libnss3 \
    libatk1.0-0 \
    libatk-bridge2.0-0 \
    libcups2 \
    libdrm2 \
    libxkbcommon0 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libgbm1 \
    libpango-1.0-0 \
    libcairo2 \
    libasound2t64 \
    libxshmfence1
```

# Rust

```bash
# 1. Installer rustup en tant que utilisateur cloudpanel
sudo su <utilisateur_cloudpanel>
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 2. Dépendances de compilation en tant que ROOT
su - root
sudo apt install build-essential pkg-config

# 3. Vérifier
sudo su <utilisateur_cloudpanel>
rustc --version
cargo --version
```
