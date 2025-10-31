# 07 – Restauración y Mantenimiento

En este paso se documenta cómo **restaurar** una copia completa del entorno de Nextcloud (local o remota) y cómo realizar **mantenimiento preventivo** del sistema y los contenedores Docker.

---

## 1. Restauración desde backup local

Los backups generados en el Paso 04 se guardan en:
```
~/docker/automation/backups/YYYY-MM-DD_HH-MM/
```

Ejemplo de contenido:
```
db.sql.zst
nextcloud_html.tar.zst
SHA256SUMS
```

### Verificar integridad del backup

```bash
cd ~/docker/automation/backups/YYYY-MM-DD_HH-MM
sha256sum -c SHA256SUMS
```

Si todos los archivos muestran `OK`, el backup es válido.

---

## 2. Restaurar base de datos (MariaDB)

Detener el contenedor actual de la base de datos:

```bash
cd ~/docker/nextcloud
docker compose stop db
```

Eliminar el contenido anterior de la base de datos (cuidado: destruye datos previos):

```bash
sudo rm -rf ~/docker/nextcloud/db/*
```

Descomprimir y restaurar el backup:

```bash
zstd -d ~/docker/automation/backups/YYYY-MM-DD_HH-MM/db.sql.zst -o /tmp/db.sql
docker compose up -d db
sleep 5
docker exec -i nextcloud-db-1 mariadb -u root -p < /tmp/db.sql
```

---

## 3. Restaurar archivos de la aplicación Nextcloud

Detener los servicios activos:

```bash
docker compose down
```

Eliminar archivos antiguos del contenedor web:

```bash
sudo rm -rf ~/docker/nextcloud/app/*
```

Restaurar el backup:

```bash
tar -xvf ~/docker/automation/backups/YYYY-MM-DD_HH-MM/nextcloud_html.tar.zst -C ~/docker/nextcloud/app --use-compress-program=unzstd
```

Levantar los servicios nuevamente:

```bash
docker compose up -d
```

---

## 4. Restaurar desde backup remoto (Windows)

Si tenés una copia off‑site enviada por Tailscale (Paso 06), podés traerla nuevamente con `scp`:

```bash
# Variables de ejemplo
WIN_USER="<usuario_windows>"
WIN_HOST="100.x.y.z"
WIN_DIR="/c/NextcloudBackups/YYYY-MM-DD_HH-MM"

# Descargar copia al Raspberry Pi
scp -r "$WIN_USER@$WIN_HOST:$WIN_DIR" ~/docker/automation/backups/
```

Luego seguir el mismo proceso de restauración local.

---

## 5. Verificar estado de Nextcloud

Entrar al contenedor de Nextcloud:

```bash
docker exec -it nextcloud-app-1 bash
```

Activar/desactivar modo mantenimiento:

```bash
sudo -u www-data php occ maintenance:mode --on
sudo -u www-data php occ maintenance:mode --off
```

Reparar estructura interna si algo falla:

```bash
sudo -u www-data php occ maintenance:repair
```

Salir del contenedor:

```bash
exit
```

---

## 6. Mantenimiento periódico

Se recomienda realizar las siguientes tareas manualmente cada cierto tiempo, además de las automatizaciones ya creadas.

### Actualizar sistema operativo

```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

### Actualizar contenedores Docker

```bash
cd ~/docker/nextcloud
docker compose pull
docker compose up -d
```

### Limpiar imágenes antiguas y espacio

```bash
docker system prune -af
sudo journalctl --vacuum-time=7d
```

### Verificar logs de automatización

```bash
cat ~/docker/automation/logs/watchdog.log
cat ~/docker/automation/logs/backup.log
cat ~/docker/automation/logs/update.log
```

---

## 7. Rotación de backups y espacio libre

Para verificar cuántos backups existen:

```bash
ls -lh ~/docker/automation/backups
```

Eliminar manualmente los más antiguos (ya automatizado en `backup_nextcloud.sh`):

```bash
find ~/docker/automation/backups -maxdepth 1 -type d -mtime +15 -exec rm -rf {} \;
```

---

## 8. Verificación de notificaciones

Asegurate de recibir correctamente los correos automáticos de:
- Backups (`backup_nextcloud.sh`)
- Actualizaciones (`update_os.sh` y `update_stack.sh`)
- Watchdog
- Alerta SSH (Paso 05)

Para probar manualmente el envío de correo:

```bash
echo "Prueba de alerta SMTP" | mail -s "Test msmtp" <tu_correo@dominio.com>
```

---

## 9. Recuperación de emergencia

### Restaurar solo base de datos
Si el contenedor de la app funciona pero hay corrupción en la base:

```bash
docker compose stop db
zstd -d ~/docker/automation/backups/YYYY-MM-DD_HH-MM/db.sql.zst -o /tmp/db.sql
docker compose up -d db
sleep 5
docker exec -i nextcloud-db-1 mariadb -u root -p < /tmp/db.sql
```

### Restaurar solo archivos de Nextcloud

```bash
docker compose down
tar -xvf ~/docker/automation/backups/YYYY-MM-DD_HH-MM/nextcloud_html.tar.zst -C ~/docker/nextcloud/app --use-compress-program=unzstd
docker compose up -d
```

### Reinstalar Docker sin perder datos

En caso extremo, podés reinstalar Docker sin perder tus volúmenes de datos (contenidos en `~/docker/nextcloud/`).  
Simplemente reinstalá Docker y vuelve a ejecutar el `docker compose up -d` desde tu carpeta del proyecto.

---

Con este procedimiento, el entorno **Nextcloud + Raspberry Pi + Automatización** queda completamente recuperable y mantenible.  
Este paso final cierra el ciclo de instalación, seguridad y respaldo del sistema, asegurando alta disponibilidad y resiliencia ante fallos.

