
---

# 📄 docs/05-automatizacion.md
```markdown
# 05 - Automatización

## Introducción
Para mantener el servidor estable, se usan scripts bash que automatizan tareas críticas:  
backups, actualizaciones, monitoreo y notificaciones.

---

## 1. Scripts disponibles (carpeta `/automation`)
- `backup-nextcloud.sh` → genera backup (archivos + DB).
- `restore-nextcloud.sh` → restaura un backup completo.
- `update-nextcloud.sh` → actualiza contenedor Nextcloud.
- `update-docker.sh` → actualiza todas las imágenes Docker.
- `update-raspi.sh` → actualiza OS y firmware.
- `notify.sh` → función genérica para enviar mails (msmtp).
- `watch-nextcloud.sh` → verifica contenedor y lo reinicia si cae.

---

## 2. Ejemplo de programación en cron
```cron
# Backup diario 03:15
15 3 * * * /home/<usuario>/automation/backup-nextcloud.sh >> /home/<usuario>/backup.log 2>&1

# Watchdog cada 10 minutos
*/10 * * * * /home/<usuario>/automation/watch-nextcloud.sh

# Actualizaciones semanales
0 5 * * 0 /home/<usuario>/automation/update-docker.sh
0 6 * * 0 /home/<usuario>/automation/update-raspi.sh

