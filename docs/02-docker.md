
## Instalar Docker y Docker Compose plugin

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

Ejemplo de docker-compose.yml

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

Ejecución

```bash
cd config
docker compose pull
docker compose up -d
docker compose ps
```
