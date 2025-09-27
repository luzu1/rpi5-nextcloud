# 05 - Automatización

## Introducción
Automatizamos tareas clave para mantener Nextcloud estable y seguro:
- Notificaciones por correo (`notify.sh`)
- Backups y restauración
- Actualizaciones (contenedores y sistema)
- Watchdog: reinicia el contenedor si cae
- Alertas de disco lleno e intentos SSH fallidos
Todas las tareas viven en `~/automation` y se programan con `cron`.

---

### 1) Preparación: carpeta de scripts y logs

```bash
mkdir -p ~/automation
mkdir -p ~/backups/nextcloud
```
"Usa un usuario no root. Los scripts leerán variables desde ~/config/.env si existe (DB_USER, DB_PASSWORD, etc.)."

----

### 2) Notificaciones por correo (notify.sh)


2.1 Instalar y configurar msmtp:

```bash
sudo apt update
sudo apt install -y msmtp msmtp-mta
```

Archivo de configuración ""(con tus datos reales; no subir al repo)"":

```bash
cat > ~/.msmtprc <<'EOF'
account default
host smtp.example.com
port 587
auth on
user usuario@example.com
password CONTRASEÑA_SEGURA
from usuario@example.com
tls on
tls_starttls on
EOF

chmod 600 ~/.msmtprc
```

2.2 Crear notify.sh

```bash
cat > ~/automation/notify.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
SUBJECT="${1:-Notificación}"
BODY="${2:-Sin contenido}"
TO="${NOTIFY_TO:-admin@example.com}"   # exporta NOTIFY_TO en tu shell para cambiar el destino
echo -e "Subject: ${SUBJECT}\n\n${BODY}" | msmtp -a default "$TO"
BASH
chmod +x
 ~/automation/notify.sh
```

Prueba:
```bash
~/automation/notify.sh "Prueba de notificación" "Esto es un test desde $(hostname)"
```

----------

### 3) Backups automáticos


3.1 Script backup-nextcloud.sh:

```bash
cat > ~/automation/backup-nextcloud.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail

BACKUP_ROOT="$HOME/backups/nextcloud"
DATA_DIR="/srv/nextcloud/data"
STAMP="$(date +%F_%H%M)"
TARGET_DIR="${BACKUP_ROOT}/${STAMP}"

# Variables de ~/config/.env (DB_NAME/USER/PASSWORD)
if [ -f "$HOME/config/.env" ]; then
  set -a
  source "$HOME/config/.env"
  set +a
fi

mkdir -p "$TARGET_DIR"

# 1) Dump DB
docker exec nextcloud_db sh -c "mysqldump -u${DB_USER} -p${DB_PASSWORD} ${DB_NAME}" \
  > "${TARGET_DIR}/db.sql"

# 2) Tar datos
tar -C "$DATA_DIR" -czf "${TARGET_DIR}/data.tgz" .

# 3) Retención (21 días)
find "$BACKUP_ROOT" -maxdepth 1 -type d -name "20*" -mtime +21 -exec rm -rf {} \;

# 4) Notificar
"$HOME/automation/notify.sh" "Backup Nextcloud ${STAMP}" "Backup en ${TARGET_DIR} OK"
BASH
chmod +x ~/automation/backup-nextcloud.sh
```

3.2 Script restore-nextcloud.sh

```bash
cat > ~/automation/restore-nextcloud.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail

BACKUP_ROOT="$HOME/backups/nextcloud"
TARGET="${1:-}"  # e.g. 2025-09-27_0315
[ -z "$TARGET" ] && { echo "Uso: $0 <YYYY-MM-DD_HHMM>"; exit 1; }

DATA_DIR="/srv/nextcloud/data"
BACKUP_DIR="${BACKUP_ROOT}/${TARGET}"

# 1) Restaurar datos
tar -C "$DATA_DIR" -xzf "${BACKUP_DIR}/data.tgz"

# 2) Restaurar DB
if [ -f "$HOME/config/.env" ]; then
  set -a; source "$HOME/config/.env"; set +a
fi
docker exec -i nextcloud_db sh -c "mysql -u${DB_USER} -p${DB_PASSWORD} ${DB_NAME}" \
  < "${BACKUP_DIR}/db.sql"

"$HOME/automation/notify.sh" "Restore Nextcloud" "Restauración desde ${TARGET} completada"
BASH
chmod +x ~/automation/restore-nextcloud.sh
````

------

### 4) Actualizaciones


4.1 Solo Nextcloud (update-nextcloud.sh)

```bash
cat > ~/automation/update-nextcloud.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
cd "$HOME/config"
docker compose pull nextcloud_app
docker compose up -d nextcloud_app
"$HOME/automation/notify.sh" "Update Nextcloud" "nextcloud_app actualizado correctamente"
BASH
chmod +x ~/automation/update-nextcloud.sh
```

4.2 Todo el stack (update-docker.sh)

```bash
cat > ~/automation/update-docker.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
cd "$HOME/config"
docker compose pull
docker compose up -d
docker image prune -f
"$HOME/automation/notify.sh" "Update Docker" "Stack actualizado y limpieza realizada"
BASH
chmod +x ~/automation/update-docker.sh
```

4.3 Sistema operativo (update-raspi.sh)

```bash
cat > ~/automation/update-raspi.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
sudo apt update && sudo apt -y upgrade && sudo apt -y autoremove
"$HOME/automation/notify.sh" "Update OS" "Sistema operativo actualizado"
BASH
chmod +x ~/automation/update-raspi.sh
```

-------

### 5) Watchdog (reinicio si cae el contenedor)

```bash
cat > ~/automation/watch-nextcloud.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
STATUS=$(docker inspect -f '{{.State.Running}}' nextcloud_app 2>/dev/null || echo "false")
if [ "$STATUS" != "true" ]; then
  docker restart nextcloud_app || true
  "$HOME/automation/notify.sh" "ALERTA: Nextcloud reiniciado" "El contenedor nextcloud_app estaba detenido y fue reiniciado"
fi
BASH
chmod +x ~/automation/watch-nextcloud.sh
```

---------

6) Alerta por disco lleno

```bash
cat > ~/automation/disk-alert.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD="${1:-85}"      # porcentaje
USED=$(df -h /srv | awk 'NR==2 {gsub("%","",$5); print $5}')
if [ "$USED" -ge "$THRESHOLD" ]; then
  "$HOME/automation/notify.sh" "Alerta: disco al ${USED}%" "El uso de /srv alcanzó ${USED}% (umbral ${THRESHOLD}%)"
fi
BASH
chmod +x ~/automation/disk-alert.sh
```

--------

### 7) Alerta por intentos SSH fallidos (últimos 15 minutos)

```bash
cat > ~/automation/ssh-alert.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
LOGS=$(journalctl -u ssh --since "-15 minutes" | grep "Failed password" || true)
if [ -n "$LOGS" ]; then
  "$HOME/automation/notify.sh" "Alerta: intentos SSH fallidos" "$LOGS"
fi
BASH
chmod +x ~/automation/ssh-alert.sh
```

-------

## 8) Programar todo con cron

```bash
# Backup diario 03:15
( crontab -l 2>/dev/null; echo '15 3 * * * /home/'"$USER"'/automation/backup-nextcloud.sh >> /home/'"$USER"'/backup.log 2>&1' ) | crontab -

# Watchdog cada 10 min
( crontab -l 2>/dev/null; echo '*/10 * * * * /home/'"$USER"'/automation/watch-nextcloud.sh' ) | crontab -

# Alerta de disco (diaria 07:00, umbral 85%)
( crontab -l 2>/dev/null; echo '0 7 * * * /home/'"$USER"'/automation/disk-alert.sh 85' ) | crontab -

# Alerta SSH cada 15 min
( crontab -l 2>/dev/null; echo '*/15 * * * * /home/'"$USER"'/automation/ssh-alert.sh' ) | crontab -

# Actualizaciones semanales (domingo)
( crontab -l 2>/dev/null; echo '0 5 * * 0 /home/'"$USER"'/automation/update-docker.sh' ) | crontab -
( crontab -l 2>/dev/null; echo '0 6 * * 0 /home/'"$USER"'/automation/update-raspi.sh' ) | crontab -
```

Ver cron:
```bash
crontab -l
```

-------

### 9) Pruebas

```bash
# Forzar un backup ahora
~/automation/backup-nextcloud.sh

# Simular caída y ver watchdog (detener y esperar 10 min o ejecutar watchdog)
docker stop nextcloud_app
~/automation/watch-nextcloud.sh

# Probar alertas
~/automation/disk-alert.sh 0
~/automation/ssh-alert.sh
```
