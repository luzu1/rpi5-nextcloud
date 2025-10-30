# 02 – Docker & Nextcloud

instalar Docker/Compose, crear la estructura del proyecto, definir variables en `.env`, crear `docker-compose.yml` y desplegar **Nextcloud + MariaDB** en el puerto **8080**.

---

## 1) Instalar Docker y Compose plugin

*en la Raspberry Pi por SSH o consola local.* : descarga e instala Docker y el plugin Compose v2. Las últimas dos líneas confirman que ambas herramientas están disponibles.

```bash
# Actualiza paquetes base
sudo apt update && sudo apt full-upgrade -y

# Instala Docker Engine (script oficial)
curl -fsSL https://get.docker.com | sh

# (Opcional) instala utilidades si faltan
sudo apt install -y ca-certificates curl gnupg lsb-release

# Verifica versiones
docker --version
docker compose version
```
---

## 2) Crear estructura de carpetas

*en la Raspberry Pi.* : crea una ruta clara para tus archivos del stack (`~/docker/nextcloud`) y una carpeta opcional para futuras copias.

```bash
# Carpeta del proyecto Nextcloud
mkdir -p ~/docker/nextcloud

# (Opcional) Carpeta para backups locales
mkdir -p ~/backups/nextcloud
```
---

## 3) Variables de entorno (.env)

*en la Raspberry Pi, dentro de la carpeta del proyecto.*

```bash
cd ~/docker/nextcloud
nano .env
```

**Ajusta:**
```ini
# Zona horaria
TZ=Europe/Madrid

# --- MariaDB ---
MYSQL_ROOT_PASSWORD=CAMBIA_ESTE_VALOR_ROOT
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
NC_DB_PASS=CAMBIA_ESTE_VALOR_DB

# --- Nextcloud ---
# Dominio público o IP que Nextcloud debe confiar (se usa dentro del contenedor)
TRUSTED_DOMAINS=cloud.example.com
```

**Qué hace:** centraliza credenciales y parámetros que usará `docker-compose.yml`. Usa contraseñas largas y únicas.

---

## 4) Crear `docker-compose.yml` (sin `version`)

**Dónde ejecutar:** *en la Raspberry Pi, en `~/docker/nextcloud`.*

```bash
nano docker-compose.yml
```

**Pega:**
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
      - "8080:80"              # Acceso local http://IP_RASPBERRY:8080
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

**Qué hace:** define dos servicios: `db` (MariaDB 11.4) y `app` (Nextcloud Apache). Los datos de la base se guardan en `./db` y los de la aplicación en el volumen `nextcloud_html`. El puerto 8080 queda publicado para acceso web local.

> **Nota:** omitimos la clave `version` para evitar advertencias con Compose v2.

---

## 5) Desplegar el stack

**Dónde ejecutar:** *en la Raspberry Pi, dentro de `~/docker/nextcloud`.*

```bash
cd ~/docker/nextcloud

# Descarga imágenes y levanta en segundo plano
docker compose pull
docker compose up -d

# Ver estado
docker compose ps
```

**Qué hace:** baja las imágenes necesarias y lanza los contenedores. `ps` debe mostrar `nextcloud_db` y `nextcloud_app` en `running`.

---

## 6) Primer acceso y asistente web

**Desde el navegador de tu PC en la LAN:**  
`http://IP_DE_TU_RASPBERRY:8080`

Si usas **Tailscale**, también funciona con la IP Tailscale (`100.x.x.x`):  
`http://IP_TAILSCALE:8080`

En el asistente web:
- Crea **usuario administrador** y contraseña.
- En **Base de datos**, elige **MySQL/MariaDB** y completa:
  - **Servidor**: `db`
  - **Usuario**: `${MYSQL_USER}`
  - **Contraseña**: `${NC_DB_PASS}`
  - **Base de datos**: `${MYSQL_DATABASE}`

**Qué hace:** finaliza la instalación de Nextcloud y enlaza con MariaDB usando los valores del `.env`.

---

## 7) Comprobaciones desde consola

**Dónde ejecutar:** *en la Raspberry Pi.*

```bash
# Comprobar estado HTTP del contenedor (debe devolver JSON con "installed")
curl -fsSL http://127.0.0.1:8080/status.php

# Estado interno con occ
docker exec -u www-data nextcloud_app php occ status

# (Por si quedó activo) desactivar modo mantenimiento
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
```

**Qué hace:** valida que la instancia está instalada y operativa.

---

## 8) Ajustes útiles (opcionales)

**Actualizar trusted domains por CLI (si cambiaste dominio/IP):**
```bash
docker exec -u www-data nextcloud_app php occ config:system:set trusted_domains 1 --value="${TRUSTED_DOMAINS}"
```

**Aplicar índices en BD (mejoras internas):**
```bash
docker exec -u www-data nextcloud_app php occ db:add-missing-indices
```

---

## 9) Actualizar contenedores (manual)

**Dónde ejecutar:** *en la Raspberry Pi, dentro del proyecto.*

```bash
cd ~/docker/nextcloud
docker compose pull
docker compose up -d
docker image prune -f
```

**Qué hace:** trae nuevas versiones y reinicia con mínima interrupción. El prune elimina capas obsoletas.

---

## 10) Solución de problemas comunes

**Error 500 o “Access denied for user 'nextcloud'@…”**  
Asegurate de que **usuario/contraseña/base** coinciden entre `.env`, `docker-compose.yml` y lo que ingresaste en el asistente web. Si cambiaste la contraseña, puedes forzarla en MariaDB:

```bash
cd ~/docker/nextcloud
# Reaplicar credenciales en MariaDB
docker exec -i nextcloud_db mariadb -u root -p"${MYSQL_ROOT_PASSWORD}" <<'SQL'
ALTER USER '${MYSQL_USER}'@'%' IDENTIFIED BY '${NC_DB_PASS}';
FLUSH PRIVILEGES;
SQL
# Reiniciar la app
docker compose restart app
```

**`status.php` vacío o 500**  
Revisa logs:
```bash
docker logs --tail=200 nextcloud_app
docker logs --tail=200 nextcloud_db
```

---

✅ Con esto queda **Nextcloud + MariaDB** operativos en Docker.  
La **exposición pública con Cloudflare Tunnel** y el **endurecimiento de seguridad** se realizan en los pasos siguientes.
