## Introducción
En este proyecto usamos **Docker** y **Docker Compose** para desplegar Nextcloud y sus servicios en una Raspberry Pi 5.  

La instalación de **Nextcloud** se hace **dentro de Docker**: no se descarga manualmente ni se configura PHP/Apache en el host.  
El servicio `nextcloud_app` en `docker-compose.yml` se encarga de levantar automáticamente Nextcloud con su configuración mínima.  


## 1) Instalar Docker y Docker Compose plugin

```bash
sudo apt update && sudo apt -y upgrade
curl -fsSL https://get.docker.com | sh
sudo apt -y install docker-compose-plugin
```
Verifica:

```bash
docker --version
docker compose version
```

Estructura de carpetas recomendada

```bash
sudo mkdir -p /srv/nextcloud/{db,app,data}
sudo mkdir -p /srv/npm/{data,letsencrypt}
sudo mkdir -p /srv/cloudflared
```

- /srv/nextcloud/db → datos de MariaDB

- /srv/nextcloud/app → configuración de Nextcloud

- /srv/nextcloud/data → archivos de usuarios

- /srv/npm/* → (opcional) Nginx Proxy Manager

- /srv/cloudflared → (opcional) configuración del túnel

----------

## 2) Variables de entorno

Crea un .env local (usa valores reales solo en el servidor). En el repo público sube .env.example con placeholders.
```bash
mkdir -p ~/config
cat > ~/config/.env <<'EOF'
# Zona horaria
TZ=Europe/Madrid

# MariaDB (Nextcloud)
DB_ROOT_PASSWORD=CAMBIA_ESTE_VALOR
DB_NAME=nextcloud
DB_USER=nc_user
DB_PASSWORD=CAMBIA_ESTE_VALOR

# Cloudflare Tunnel (opcional)
CLOUDFLARED_TUNNEL_TOKEN=PEGA_AQUI_TU_TOKEN
EOF
```
Proteger con permisos:
```bash
chmod 600 ~/config/.env
```

-----------

## 3) Archivo `docker-compose.yml`

Aquí definimos **todos los servicios**.  
El contenedor `nextcloud_app` es el que instala y corre Nextcloud:

```bash
version: "3.8"

services:
  nextcloud_db:
    image: mariadb:10.11
    container_name: nextcloud_db
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MYSQL_DATABASE=${DB_NAME}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASSWORD}
      - TZ=${TZ}
    volumes:
      - /srv/nextcloud/db:/var/lib/mysql

  nextcloud_app:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud_app
    depends_on:
      - nextcloud_db
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${TZ}
    volumes:
      - /srv/nextcloud/app:/config
      - /srv/nextcloud/data:/data
    ports:
      - "8080:80"   # acceso LAN; el acceso externo se hace por túnel

  reverse_proxy:
    image: jc21/nginx-proxy-manager:latest
    container_name: reverse_proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - /srv/npm/data:/data
      - /srv/npm/letsencrypt:/etc/letsencrypt

  tunnel:
    image: cloudflare/cloudflared:latest
    container_name: tunnel
    restart: unless-stopped
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARED_TUNNEL_TOKEN}
    command: tunnel run
```
--------
## 4) Ejecución

```bash
cd config
docker compose pull
docker compose up -d
docker compose ps
```

----------

## 5) Acceso:

- LAN: http://<IP-LAN>:8080

- Externo: el dominio configurado en tu túnel (si usas tunnel).

------

## 6) Primeros ajustes de Nextcloud:

- Si el instalador indica problemas de permisos en data, corrige:
```bash
sudo chown -R 1000:1000 /srv/nextcloud/app /srv/nextcloud/data
docker restart nextcloud_app
```

------

## 7) Troubleshooting rápido

- Ver contenedores activos
```bash
docker ps
```
- Últimas líneas de logs
```bash
docker logs -n 200 nextcloud_app
docker logs -n 200 nextcloud_db
```
- Ver puertos en uso
```bash
ss -tulpn | grep -E '(:80|:81|:443|:8080)'
```
- Revisión de permisos si Nextcloud no puede escribir
```bash
sudo chown -R 1000:1000 /srv/nextcloud/app /srv/nextcloud/data
```
