# 05 - Backups

## Introducción
Para garantizar la seguridad de los datos, es fundamental implementar un sistema de **copias de seguridad automáticas**.  
En este proyecto los backups incluyen:

- Dump completo de la base de datos **MariaDB**  
- Archivos de usuario en `/srv/nextcloud/data`  
- Retención de **21 días para diarios** y **4 semanas para semanales**  
- Notificación automática por correo al finalizar  

Los backups se almacenan en el propio servidor (ej: `/home/<usuario>/backups/nextcloud`).  
Se recomienda sincronizarlos periódicamente con un almacenamiento externo (NAS, Nextcloud externo o S3).

---

1) Crear directorio para backups

```bash
mkdir -p /home/$USER/backups/nextcloud
```
2) Script de backup
```bash
mkdir -p ~/automation
nano ~/automation/backup-nextcloud.sh
```

Contenido de backup-nextcloud.sh
```bash
#!/usr/bin/env bash
set -euo pipefail

# Directorios
BACKUP_ROOT="/home/$USER/backups/nextcloud"
DATA_DIR="/srv/nextcloud/data"
DATE="$(date +%F_%H%M)"

# Cargar variables de entorno desde ~/config/.env
if [ -f "$HOME/config/.env" ]; then
  set -a
  source "$HOME/config/.env"
  set +a
fi

mkdir -p "${BACKUP_ROOT}/${DATE}"

# 1) Dump de la base de datos
docker exec nextcloud_db sh -c "mysqldump -u${DB_USER} -p${DB_PASSWORD} ${DB_NAME}" \
  > "${BACKUP_ROOT}/${DATE}/db.sql"

# 2) Archivar los datos de Nextcloud
tar -C "${DATA_DIR}" -czf "${BACKUP_ROOT}/${DATE}/data.tgz" .

# 3) Rotar copias antiguas (21 días)
find "${BACKUP_ROOT}" -maxdepth 1 -type d -name "20*" -mtime +21 -exec rm -rf {} \;

# 4) Notificar (usa notify.sh si está configurado)
if [ -x "$HOME/automation/notify.sh" ]; then
  "$HOME/automation/notify.sh" "Backup Nextcloud ${DATE}" "Backup guardado en ${BACKUP_ROOT}/${DATE}"
fi
```
Dar permisos de ejecucion:
```bash
chmod +x ~/automation/backup-nextcloud.sh
```
---------------
3) Script de restauración
Este script permite restaurar una copia específica.
```bash
nano ~/automation/restore-nextcloud.sh
```
Contenido de restore-nextcloud.sh
```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_ROOT="/home/$USER/backups/nextcloud"
TARGET="${1:-}"

if [ -z "$TARGET" ]; then
  echo "Uso: $0 <YYYY-MM-DD_HHMM>"
  exit 1
fi

DATA_DIR="/srv/nextcloud/data"

# 1) Restaurar archivos
sudo tar -C "${DATA_DIR}" -xzf "${BACKUP_ROOT}/${TARGET}/data.tgz"

# 2) Restaurar base de datos
if [ -f "$HOME/config/.env" ]; then
  set -a; source "$HOME/config/.env"; set +a
fi

docker exec -i nextcloud_db sh -c "mysql -u${DB_USER} -p${DB_PASSWORD} ${DB_NAME}" \
  < "${BACKUP_ROOT}/${TARGET}/db.sql"

echo "Restauración completada desde ${TARGET}"
```

Dar permisos:
```bash
chmod +x ~/automation/restore-nextcloud.sh
```

prueba:
```bash
./restore-nextcloud.sh 2025-09-27_0315
```
-----------

4) Programación con cron

Editar el cron del usuario
```bash
crontab -e
```
Agregar las tareas (Backup diario a las 03:15)
```bash
15 3 * * * /home/$USER/automation/backup-nextcloud.sh >> /home/$USER/backup.log 2>&1
```
-------
5) Comprobación

Verificar que se generan las carpetas con fecha:
```bash
ls -lh /home/$USER/backups/nextcloud/
```
Comprobar el contenido:
```bash
du -sh /home/$USER/backups/nextcloud/*
```
