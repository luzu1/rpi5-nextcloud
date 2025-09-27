# 04 - Seguridad

## Introducción
Un servidor expuesto a Internet requiere una configuración mínima de seguridad.  
Aquí aplicamos buenas prácticas en:
- **Acceso SSH**
- **Firewall UFW**
- **Exposición de servicios**
- **Alertas automáticas**

---

### 1) Acceso SSH seguro


a) Deshabilitar contraseñas
Edita la configuración de SSH:

```bash
sudo nano /etc/ssh/sshd_config
```
Cambia estas opciones:
```bash
PasswordAuthentication no
PermitRootLogin prohibit-password
```
Reinicia el servicio:
```bash
sudo systemctl restart ssh
```


b) Claves públicas

Asegúrate de tener tu clave pública cargada en el servidor:
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```


c) Restringir acceso por Tailscale

Permitir SSH solo por la interfaz tailscale0:
```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

-----------

### 2) Firewall con UFW


a) Política por defecto
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```


b) Permitir solo tráfico necesario

- SSH vía Tailscale

- HTTP/HTTPS solo desde la LAN
```bash
sudo ufw allow from 192.168.1.0/24 to any port 80,443 proto tcp
```


c) Activar el firewall
```bash
sudo ufw enable
sudo ufw status verbose
```

------

### 3) Exposición de servicios

- No abrir puertos en el router.

- Cloudflare Tunnel gestiona la exposición pública de Nextcloud.

- El DNS del dominio debe apuntar a Cloudflare con proxy activado (nube naranja).

- Certificados TLS son gestionados por Cloudflare, no directamente por la Raspberry.

--------

### 4) Alertas automáticas

Todas las notificaciones se centralizan con notify.sh, un script que usa msmtp para enviar correos.


a) Configurar msmtp


Instalar msmtp:
```bash
sudo apt install -y msmtp msmtp-mta
```
Crear configuración local:
```bash
nano ~/.msmtprc
```
ejemplo:
```bash
account default
host smtp.example.com
port 587
auth on
user usuario@example.com
password CONTRASEÑA_SEGURA
from usuario@example.com
tls on
tls_starttls on
```

proteger archivo:
```bash
chmod 600 ~/.msmtprc
```


b) Crear script notify.sh
```bash
mkdir -p ~/automation
nano ~/automation/notify.sh
```

Contenido:
```bash
#!/usr/bin/env bash
set -euo pipefail

SUBJECT="${1:-Notificación}"
BODY="${2:-Sin contenido}"
TO="${NOTIFY_TO:-admin@example.com}"

echo -e "Subject: ${SUBJECT}\n\n${BODY}" | msmtp -a default "$TO"
```

Dar permisos:
```bash
chmod +x ~/automation/notify.sh
```

Prueba:
```bash
~/automation/notify.sh "Prueba de notificación" "Esto es un test"
```

-----

### 5) Alertas automáticas


a) Backups y actualizaciones

Todos los scripts (backup-nextcloud.sh, update-docker.sh, etc.) llaman a notify.sh al finalizar.
Esto permite recibir un correo con el resultado.


b) Contenedores caídos (watchdog)

El script watch-nextcloud.sh revisa si el contenedor nextcloud_app está activo.
Si no lo está:

-Lo reinicia con docker restart

-Envía alerta con notify.sh

Ejemplo de ejecución cada 10 minutos (cron):
```bash
*/10 * * * * /home/$USER/automation/watch-nextcloud.sh
```


c) Uso de disco alto

Un script puede revisar periódicamente el porcentaje de uso:
```bash
df -h /srv | awk 'NR==2 {print $5}'
```
Si supera un umbral (ej: 85%), llama a:
```bash
~/automation/notify.sh "Alerta: Disco casi lleno" "El uso de disco supera el 85%"
```


d) Intentos fallidos de SSH

Creamos un servicio que monitorea journalctl en busca de intentos fallidos.

Script ssh-alert.sh:
```bash
nano ~/automation/ssh-alert.sh
```
Contenido:
```bash
#!/usr/bin/env bash
set -euo pipefail

LOGS=$(journalctl -u ssh -n 50 | grep "Failed password")

if [ -n "$LOGS" ]; then
  ~/automation/notify.sh "Alerta: intentos SSH fallidos" "$LOGS"
fi
```

Dar permisos:
```bash
chmod +x ~/automation/ssh-alert.sh
```

Programar Cron cada 15 minutos:
```bash
*/15 * * * * /home/$USER/automation/ssh-alert.sh
```





