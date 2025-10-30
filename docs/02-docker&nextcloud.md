# 02 – Docker & Nextcloud

**Objetivo:** instalar Docker/Compose, definir variables en `.env`, crear `docker-compose.yml` **(sin `version`)** y desplegar **Nextcloud + MariaDB** en el puerto **8080**. *La seguridad y automatizaciones se documentan en pasos posteriores.*

---

## 1) Instalar Docker y Compose plugin

**Ejecutar en:** Raspberry Pi por SSH o consola local.

```bash
sudo apt update && sudo apt full-upgrade -y
curl -fsSL https://get.docker.com | sh
sudo apt install -y ca-certificates curl gnupg lsb-release

docker --version
docker compose version
```

Instala Docker Engine y el plugin Compose v2. Las últimas dos líneas confirman que ambas herramientas están disponibles.

---

## 2) Verificar estructura de carpetas existente

**Ejecutar en:** Raspberry Pi.

Si seguiste el paso anterior (Infraestructura), ya tendrás creadas las carpetas base. Verifica que estén presentes:

```bash
ls -R ~/docker/nextcloud
```

---

## 3) Variables de entorno (.env)

**Ejecutar en:** Raspberry Pi, dentro de la carpeta del proyecto.

```bash
cd ~/docker/nextcloud
nano .env
```

Contenido del archivo:

```ini
# Zona horaria
TZ=Europe/Madrid

# --- MariaDB ---
MYSQL_ROOT_PASSWORD=CAMBIA_ESTE_VALOR_ROOT
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
NC_DB_PASS=CAMBIA_ESTE_VALOR_DB

# --- Nextcloud ---
TRUSTED_DOMAINS=cloud.example.com
```

Centraliza credenciales y parámetros que usará `docker-compose.yml`. Usa contraseñas largas y únicas.

---

## 4) Crear docker-compose.yml (sin version)

**Ejecutar en:** Raspberry Pi, dentro de `~/docker/nextcloud`.

```bash
nano docker-compose.yml
```

Contenido del archivo:

```yaml
services:
  db:
    image: mariadb:11.4
    container_name: nextcloud_db
    restart: unless-stopped
    command: >
      --transaction-isolation=READ-COMMITTED
      --binlog-format=ROW
      --innodb_flush_log_at_trx_commit=1
      --innodb_buffer_pool_size=256M
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${NC_DB_PASS}
      - TZ=${TZ}
    volumes:
      - ./db:/var/lib/mysql

  app:
    image: nextcloud:31-apache
    container_name: nextcloud_app
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "8080:80"
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${NC_DB_PASS}
      - TRUSTED_DOMAINS=${TRUSTED_DOMAINS}
      - TZ=${TZ}
    volumes:
      - nextcloud_html:/var/www/html

volumes:
  nextcloud_html:
```

Define dos servicios: `db` (MariaDB 11.4) y `app` (Nextcloud Apache). Los datos de la base se guardan en `./db` y los de la aplicación en el volumen `nextcloud_html`. El puerto 8080 queda publicado para acceso web local.

---

## 5) Desplegar el stack

**Ejecutar en:** Raspberry Pi, dentro de `~/docker/nextcloud`.

```bash
cd ~/docker/nextcloud
docker compose pull
docker compose up -d
docker compose ps
```

Descarga las imágenes y lanza los contenedores. `ps` debe mostrar `nextcloud_db` y `nextcloud_app` en estado `running`.

---

## 6) Primer acceso y asistente web

Desde el navegador en la red local:  
`http://IP_DE_TU_RASPBERRY:8080`

Si usas Tailscale:  
`http://IP_TAILSCALE:8080`

En el asistente web:
- Crea un **usuario administrador**.
- En la sección de base de datos, selecciona **MySQL/MariaDB** y completa con:
  - Servidor: `db`
  - Usuario: `${MYSQL_USER}`
  - Contraseña: `${NC_DB_PASS}`
  - Base de datos: `${MYSQL_DATABASE}`

Finaliza la instalación.

---

## 7) Comprobaciones desde consola

**Ejecutar en:** Raspberry Pi.

```bash
curl -fsSL http://127.0.0.1:8080/status.php
docker exec -u www-data nextcloud_app php occ status
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
```

Valida que la instancia esté instalada y activa.

---

## 8) Ajustes útiles (opcionales)

**Actualizar dominio confiable:**
```bash
docker exec -u www-data nextcloud_app php occ config:system:set trusted_domains 1 --value="${TRUSTED_DOMAINS}"
```

**Agregar índices de base de datos:**
```bash
docker exec -u www-data nextcloud_app php occ db:add-missing-indices
```

---

## 9) Actualizar contenedores (manual)

**Ejecutar en:** Raspberry Pi, dentro del proyecto.

```bash
cd ~/docker/nextcloud
docker compose pull
docker compose up -d
docker image prune -f
```

Descarga versiones nuevas y elimina imágenes antiguas.

---

## 10) Solución de problemas comunes

**Error 500 o “Access denied for user 'nextcloud'@…”**  
Verifica que los parámetros de base de datos sean iguales en `.env`, `docker-compose.yml` y el asistente web.  
Si cambiaste la contraseña, puedes forzarla:

```bash
cd ~/docker/nextcloud
docker exec -i nextcloud_db mariadb -u root -p"${MYSQL_ROOT_PASSWORD}" <<'SQL'
ALTER USER '${MYSQL_USER}'@'%' IDENTIFIED BY '${NC_DB_PASS}';
FLUSH PRIVILEGES;
SQL
docker compose restart app
```

**status.php vacío o 500**  
Revisa los registros:
```bash
docker logs --tail=200 nextcloud_app
docker logs --tail=200 nextcloud_db
```

---

Nextcloud + MariaDB quedan operativos en Docker.  
