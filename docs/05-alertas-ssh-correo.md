# 05 – Alertas SSH

Configuraremos para enviar un correo cada vez que alguien inicia sesión por **SSH** en la Raspberry Pi, evitando duplicados y manteniendo registros.  
Se asume que ya configuramos **msmtp** y probaste envíos (Paso 03 – Seguridad).

---

## 1) Dependencias y verificación de correo

```bash
sudo apt update
sudo apt install -y msmtp-mta mailutils
echo "Test SMTP" | mail -s "SSH Alerts: prueba inicial" tu-correo@ejemplo.com
```

Debe llegarte un correo de prueba. Si no llega, revisa la configuración de msmtp del Paso 03.

---

## 2) Script de alerta (recomendado: PAM)

Crearemos un script que tomará datos del inicio de sesión y enviará un correo **una sola vez** por sesión SSH.

```bash
sudo nano /usr/local/bin/ssh-login-alert
```

Contenido del script:

```bash
#!/usr/bin/env bash
# ssh-login-alert: envía un email al abrir sesión SSH vía PAM (sin duplicados)

set -Eeuo pipefail

# === Configuración mínima ===
MAIL_TO="alertas@ejemplo.com"         # destino de alertas
MAIL_FROM="notificaciones@tu-dominio.com" # remitente (definido en msmtp)

# === Datos de sesión/PAM ===
USER_NAME="${PAM_USER:-$(whoami)}"
REMOTE_IP="${PAM_RHOST:-${SSH_CONNECTION%% *}}"
HOSTNAME_FQDN="$(hostname -f 2>/dev/null || hostname)"
DATE_NOW="$(date '+%Y-%m-%d %H:%M:%S %Z')"

# Evitar alertas si no es sesión remota (p. ej., consola local sin IP)
if [[ -z "${REMOTE_IP:-}" || "${REMOTE_IP}" == "127.0.0.1" || "${REMOTE_IP}" == "::1" ]]; then
  exit 0
fi

# Evitar duplicados: cache por PID de sesión y timestamp corto
CACHE_DIR="/run/ssh-login-alert"
mkdir -p "$CACHE_DIR"
MARKER="$CACHE_DIR/${PAM_TTY:-tty}-$(id -u ${USER_NAME})-${REMOTE_IP}"
if [[ -f "$MARKER" ]]; then
  # Si el marcador es reciente (<120s), no reenviar
  if [[ $(( $(date +%s) - $(stat -c %Y "$MARKER" 2>/dev/null || echo 0) )) -lt 120 ]]; then
    exit 0
  fi
fi
date +%s > "$MARKER"

# Mensaje
read -r -d '' BODY <<EOF || true
Nuevo login SSH en ${HOSTNAME_FQDN}
Fecha: ${DATE_NOW}
Usuario: ${USER_NAME}
Desde:  ${REMOTE_IP}
TTY:    ${PAM_TTY:-N/A}
EOF

# Envío por sendmail (msmtp)
{
  echo "To: ${MAIL_TO}"
  echo "From: ${MAIL_FROM}"
  echo "Subject:  SSH login alert - ${HOSTNAME_FQDN}"
  echo
  echo "${BODY}"
} | /usr/sbin/sendmail -t || logger -t ssh-login-alert "Fallo al enviar correo"
```

Permisos y propiedad:

```bash
sudo chmod 755 /usr/local/bin/ssh-login-alert
sudo chown root:root /usr/local/bin/ssh-login-alert
```

---

## 3) Integración con SSH mediante PAM (único envío por sesión)

Activa el hook de **PAM** para que el script se ejecute cuando se abre una sesión SSH.

```bash
sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak
sudo nano /etc/pam.d/sshd
```

Agrega **al final** del archivo (una línea por debajo de las existentes):

```
# SSH login alert (envía mail con msmtp)
session optional pam_exec.so seteuid /usr/local/bin/ssh-login-alert
```

Guarda y reinicia el servicio SSH:

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

Prueba desde otra máquina (o nueva sesión) y verificá que recibís **un** correo por login.

---


## 4) Pruebas y solución de problemas

Comandos útiles:

```bash
# Ver últimos correos en el log de msmtp (si está habilitado)
sudo tail -n 100 /var/log/msmtp.log

# Probar el script de forma manual simulando PAM
sudo PAM_USER="$USER" PAM_RHOST="1.2.3.4" PAM_TTY="pts/9" /usr/local/bin/ssh-login-alert

# Reiniciar servicio ssh si cambiaste PAM o profile.d
sudo systemctl restart ssh
```

Problemas comunes:
- **No llegan correos**: revisa msmtp (`/etc/msmtprc`), credenciales SMTP y la conectividad a `smtp.<proveedor>:587`.
- **Doble alerta**: usa el **método PAM** (paso 3) y elimina cualquier script en `/etc/profile.d` que haga lo mismo.
- **Permisos**: asegurate de que `/usr/local/bin/ssh-login-alert` tenga permisos ejecutables y sea propiedad de `root`.

---

## Resultado

Queda un sistema de **alertas de inicio de sesión SSH** robusto y sin duplicados, apoyado en **PAM** y en msmtp para el envío de emails.  
Esto complementa la seguridad del Paso 03 y deja todo listo para integrarlo con futuras automatizaciones.



   
