# 04 – Automatización

**Objetivo:** crear una capa de automatización para **Nextcloud en Docker** con tareas de: *watchdog*, *backup semanal*, *actualización del stack*, *actualización del sistema* y *notificaciones por email*.  
Se parte de los pasos previos: infraestructura lista, Nextcloud + MariaDB en Docker, msmtp funcional (seguridad).

---

## 1) Estructura y variables de automatización

Crea la estructura y el archivo de variables específico de automatización.

```bash
mkdir -p ~/docker/automation/{scripts,logs,backups}
nano ~/docker/automation/.env-automation
```

Contenido de `.env-automation` (ajustar valores marcados):

```ini
# === Rutas / configuración del stack ===
STACK_DIR="$HOME/docker/nextcloud"             # carpeta donde está docker-compose.yml
COMPOSE_FILE="$HOME/docker/nextcloud/docker-compose.yml"

# Nombres de contenedores según docker-compose.yml
NEXTCLOUD_CONTAINER="nextcloud_app"
DB_TYPE="mariadb"
DB_CONTAINER="nextcloud_db"

# Parámetros de base de datos usados en el dump/restore
DB_NAME="nextcloud"
DB_USER="nextcloud"
DB_PASSWORD="CAMBIA_ESTE_VALOR_DB"            # igual que NC_DB_PASS del .env del paso 02

# === Backups y logs ===
BACKUP_DIR="$HOME/docker/automation/backups"
LOG_DIR="$HOME/docker/automation/logs"
RETEN_SEMANAS="8"

# === Comprobación HTTP ===
DOMAIN_PUBLICO="https://cloud.ejemplo.com"     # dominio que apunta a Nextcloud (Cloudflare Tunnel)

# === Email (msmtp configurado en paso 03) ===
MAIL_TO="admin@ejemplo.com"
MAIL_FROM="alertas@ejemplo.com"
```

Este archivo centraliza rutas, contenedores y credenciales **solo para automatización**. La contraseña debe coincidir con la del paso 02.

---

## 2) Librería común (funciones compartidas)

Archivo: `~/docker/automation/scripts/00-common.sh`

```bash
nano ~/docker/automation/scripts/00-common.sh
```

Contenido:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# Raíz de automatización (…/automation)
ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
source "$ROOT_DIR/.env-automation"

# Rutas y logs
mkdir -p "$LOG_DIR"
STAMP="$(date '+%Y-%m-%d_%H-%M')"
LOG_FILE="$LOG_DIR/${0##*/}.log"

# Aliases/shortcuts
dc() { docker compose -f "$COMPOSE_FILE" "$@"; }

log() { printf '[%(%Y-%m-%d %H:%M:%S%z)T] %s
' -1 "$*" | tee -a "$LOG_FILE"; }

# Exclusión mutua por lockfile
acquire_lock() {
  LOCK="$ROOT_DIR/.lock.${0##*/}"
  if [ -e "$LOCK" ]; then
    log "Lock detectado ($LOCK). Saliendo."
    exit 0
  fi
  trap 'rm -f "$LOCK"' EXIT
  : > "$LOCK"
}

# Enviar correo con msmtp/mailutils ya configurado (Paso 03)
mail_send() {
  local subject="$1"; shift || true
  local body="${1:-}"
  {
    echo "To: $MAIL_TO"
    echo "From: $MAIL_FROM"
    echo "Subject: $subject"
    echo
    [ -n "$body" ] && echo "$body"
  } | /usr/sbin/sendmail -t || true
}
```

Provee funciones reutilizables: logging, lockfile, alias `dc` para `docker compose` y envío de correos.

---

## 3) Watchdog (salud del stack + autorecuperación)

Archivo: `~/docker/automation/scripts/watchdog.sh`

```bash
nano ~/docker/automation/scripts/watchdog.sh
```

Contenido:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
source "$(dirname "$0")/00-common.sh"
acquire_lock

FAILED=""

log "Watchdog: comprobando salud de contenedores..."

# Contenedores unhealthy o Exited
BAD=$(docker ps --format '{{.Names}} {{.Status}}' | awk '/(unhealthy|Exited)/ {print $0}')
[ -n "$BAD" ] && { log "Contenedores con problemas:\n$BAD"; FAILED="health"; }

# Nextcloud responde OK en /status.php
STATUS_URL="$DOMAIN_PUBLICO/status.php"
if ! curl -m 10 -fsSL "$STATUS_URL" | jq -e '.installed == true' >/dev/null 2>&1; then
  log "Nextcloud no responde correctamente en $STATUS_URL"
  FAILED="${FAILED:+$FAILED,}nc_http"
fi

# Recuperación + aviso
if [ -n "$FAILED" ]; then
  log "Intentando recuperar stack..."
  dc up -d --remove-orphans || true
  sleep 10
  if ! curl -m 10 -fsSL "$STATUS_URL" | jq -e '.installed == true' >/dev/null 2>&1; then
    log "Nextcloud aún con problemas tras reinicio."
  else
    log "Recuperado tras reinicio."
  fi
  mail_send "Nextcloud watchdog: incidente ($FAILED)" "$(tail -n 120 "$LOG_FILE")"
else
  log "OK: salud correcta."
fi
```

Comprueba contenedores y el endpoint `status.php`. Si falla, intenta **autorecuperación** y notifica por email con el log adjunto.

---

## 4) Backup semanal (BD + ficheros Nextcloud)

Archivo: `~/docker/automation/scripts/backup_nextcloud.sh`

```bash
nano ~/docker/automation/scripts/backup_nextcloud.sh
```

Contenido:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
source "$(dirname "$0")/00-common.sh"
acquire_lock

DEST_DIR="$BACKUP_DIR/$STAMP"
mkdir -p "$DEST_DIR"

log "=== Backup Nextcloud @ $STAMP ==="

# 1) Mantenimiento ON
log "Habilitando modo mantenimiento…"
docker exec -u www-data "$NEXTCLOUD_CONTAINER" php occ maintenance:mode --on || true

# 2) Dump de base de datos
log "Detectando binario de dump en el contenedor DB…"
DUMP_BIN="/usr/bin/mariadb-dump"
if ! docker exec "$DB_CONTAINER" sh -lc "command -v mariadb-dump >/dev/null 2>&1"; then
  DUMP_BIN="/usr/bin/mysqldump"
fi
log "Usando: $DUMP_BIN"

DB_DUMP="$DEST_DIR/db.sql"
log "Volcando base de datos…"
if ! docker exec "$DB_CONTAINER" sh -lc "$DUMP_BIN -u'$DB_USER' -p'$DB_PASSWORD' '$DB_NAME'" > "$DB_DUMP".tmp 2>"$DB_DUMP".err; then
  log "Error durante el dump de la BD. Detalles:"
  tail -n 50 "$DB_DUMP".err || true
  rm -f "$DB_DUMP".tmp
  docker exec -u www-data "$NEXTCLOUD_CONTAINER" php occ maintenance:mode --off || true
  mail_send "Backup Nextcloud FALLÓ (error dump BD)" "Revisar $LOG_FILE"
  exit 1
fi
if [ ! -s "$DB_DUMP".tmp ]; then
  log "Dump de BD vacío."
  rm -f "$DB_DUMP".tmp
  docker exec -u www-data "$NEXTCLOUD_CONTAINER" php occ maintenance:mode --off || true
  mail_send "Backup Nextcloud FALLÓ (dump vacío)" "Revisar $LOG_FILE"
  exit 1
fi
mv "$DB_DUMP".tmp "$DB_DUMP"
rm -f "$DB_DUMP".err

# 3) Empaquetar ficheros de Nextcloud
log "Empaquetando ficheros de Nextcloud…"
if ! docker exec "$NEXTCLOUD_CONTAINER" bash -lc 'tar -C /var/www -cf - html'   | zstd -T2 -10 -o "$DEST_DIR/nextcloud_html.tar.zst"; then
  log "Error empaquetando /var/www/html"
  docker exec -u www-data "$NEXTCLOUD_CONTAINER" php occ maintenance:mode --off || true
  mail_send "Backup Nextcloud FALLÓ (tar html)" "Error al crear nextcloud_html.tar.zst"
  exit 1
fi

# 4) Comprimir dump y checksums
zstd -T2 -10 "$DB_DUMP"
rm -f "$DB_DUMP"
( cd "$DEST_DIR" && sha256sum * > SHA256SUMS )

# 5) Mantenimiento OFF
log "Deshabilitando modo mantenimiento…"
docker exec -u www-data "$NEXTCLOUD_CONTAINER" php occ maintenance:mode --off || true

# 6) Rotación (semanas)
log "Rotación: conservando $RETEN_SEMANAS semanas."
find "$BACKUP_DIR" -maxdepth 1 -type d -printf "%P\n" | awk 'NF' | sort | head -n -"${RETEN_SEMANAS}" 2>/dev/null | while read -r old; do
  [ -n "$old" ] && rm -rf "$BACKUP_DIR/$old"
done || true

# 7) Resumen + email
SIZE=$(du -sh "$DEST_DIR" | awk '{print $1}')
mail_send "Backup Nextcloud OK - $STAMP ($SIZE)" "Backup en: $DEST_DIR

Contenido:
- nextcloud_html.tar.zst
- db.sql.zst
- SHA256SUMS

Tamaño total: $SIZE"

log "Backup finalizado en $DEST_DIR ($SIZE)."
```

Crea un backup **consistente** con mantenimiento ON/OFF, exporta BD, empaqueta `/var/www/html`, comprime, genera checksums y rota carpetas antiguas.

### (Opcional) Copia off-site al equipo Windows por Tailscale

**Opción A – `scp` (simple, no incremental):**
```bash
# Variables de destino (ajustar)
WIN_USER="usuario_windows"
WIN_HOST="100.x.x.x"                 # IP Tailscale del PC Windows
WIN_DIR="/c/NextcloudBackups"        # Carpeta en Windows

# Crear carpeta destino y subir el último backup
ssh "$WIN_USER@$WIN_HOST" "mkdir -p '$WIN_DIR'"
scp -r "$BACKUP_DIR/$STAMP" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```

**Opción B – `rsync` (incremental, requiere rsync remoto):**
```bash
# Réplica completa de la carpeta de backups
rsync -avh --delete "$BACKUP_DIR/" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```

---

## 5) Actualización del stack Docker (Nextcloud + DB)

Archivo: `~/docker/automation/scripts/update_stack.sh`

```bash
nano ~/docker/automation/scripts/update_stack.sh
```

Contenido:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
source "$(dirname "$0")/00-common.sh"
acquire_lock

OUT=$(mktemp)

log "=== Update stack (Docker compose) ==="
{
  dc pull
  dc up -d
  docker image prune -f
} &> "$OUT" || true

mail_send "Update Docker/Nextcloud completado" "$(tail -n 200 "$OUT")"
rm -f "$OUT"
```

Obtiene nuevas imágenes, recrea contenedores y limpia imágenes obsoletas. Envía un resumen por correo.

---

## 6) Actualización del sistema operativo

Archivo: `~/docker/automation/scripts/update_os.sh`

```bash
nano ~/docker/automation/scripts/update_os.sh
```

Contenido:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
source "$(dirname "$0")/00-common.sh"
acquire_lock

OUT=$(mktemp)
{
  sudo apt-get update -y
  sudo DEBIAN_FRONTEND=noninteractive apt-get -y dist-upgrade
  sudo apt-get -y autoremove --purge
  sudo apt-get -y autoclean
} &> "$OUT" || true

if [ -f /var/run/reboot-required ]; then
  log "Reboot requerido (manual)."
fi

mail_send "Update OS completado" "$(tail -n 200 "$OUT")"
rm -f "$OUT"
```

Actualiza paquetes del sistema y notifica por correo. Si requiere reinicio, lo deja indicado en el log.

---

## 7) Permisos y pruebas iniciales

Dar permisos de ejecución y probar manualmente.

```bash
chmod +x ~/docker/automation/scripts/*.sh

# Pruebas
bash ~/docker/automation/scripts/watchdog.sh
bash ~/docker/automation/scripts/backup_nextcloud.sh
bash ~/docker/automation/scripts/update_stack.sh
bash ~/docker/automation/scripts/update_os.sh
```

Verifica que lleguen correos y que se creen directorios de backup con contenido.

---

## 8) Programación con cron

Abrir el cron del usuario y registrar tareas:

```bash
crontab -e
```

Entradas:

```cron
# PATH para cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Watchdog cada 5 minutos
*/5 * * * * /usr/bin/env bash $HOME/docker/automation/scripts/watchdog.sh >> $HOME/docker/automation/logs/watchdog.log 2>&1

# Backup semanal (domingo 03:30)
30 3 * * 0 /usr/bin/env bash $HOME/docker/automation/scripts/backup_nextcloud.sh >> $HOME/docker/automation/logs/backup.log 2>&1

# Update del stack Docker diario (04:30)
30 4 * * * /usr/bin/env bash $HOME/docker/automation/scripts/update_stack.sh >> $HOME/docker/automation/logs/update_stack.log 2>&1
```

Para **update del sistema** se recomienda usar el **cron de root** (no pide contraseña):

```bash
sudo crontab -e
```

Entrada sugerida:

```cron
# Update OS semanal (sábado 04:00)
0 4 * * 6 /usr/bin/env bash /home/<usuario>/docker/automation/scripts/update_os.sh >> /home/<usuario>/docker/automation/logs/update_os.log 2>&1
```

Reemplazar `<usuario>` por el nombre real del usuario en el sistema.

---

## 9) Verificación y mantenimiento

Comandos útiles:

```bash
# Ver logs recientes
tail -n 200 ~/docker/automation/logs/*.log

# Ver directorios de backup
ls -lh ~/docker/automation/backups/*/

# Ver estado del stack
docker compose -f ~/docker/nextcloud/docker-compose.yml ps
```

---

## 10) Restauración rápida (referencia)

**Restaurar ficheros de Nextcloud:**

```bash
# Parar app (mantener DB en marcha)
docker compose -f ~/docker/nextcloud/docker-compose.yml stop app

# Descomprimir contenido de html
zstd -d -c ~/docker/automation/backups/AAAA-MM-DD_HH-MM/nextcloud_html.tar.zst |   docker exec -i nextcloud_app tar -C /var/www -xf -

# Arrancar app
docker compose -f ~/docker/nextcloud/docker-compose.yml start app
```

**Restaurar base de datos:**

```bash
zstd -d -c ~/docker/automation/backups/AAAA-MM-DD_HH-MM/db.sql.zst |   docker exec -i nextcloud_db mariadb -u nextcloud -p'LA_CONTRASEÑA_DB' nextcloud
```

Asegurarse de usar la contraseña correcta y el conjunto de archivos correspondientes.

---

Con esto queda lista la **automatización base** de Nextcloud: salud supervisada, backups semanales, actualización controlada y alertas por correo. En proyectos reales se recomienda complementar con copias **off-site** y pruebas periódicas de restauración.
