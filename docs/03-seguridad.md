# 03 - Seguridad

## Introducción

En este paso se configurará la **seguridad base** del servidor Raspberry Pi 5 con **Ubuntu Server 24.04 LTS**.  
Estas medidas aseguran el acceso controlado al sistema, cifran las conexiones y limitan la exposición a la red.

El entorno incluye:
- Acceso remoto mediante **Tailscale** (sin puertos abiertos).
- Control de acceso por **claves SSH** (sin contraseñas).
- **Firewall UFW** para restringir servicios.
- **Alertas de inicio de sesión SSH** por correo (vía msmtp/Hostinger SMTP).
- DNS público administrado con **Cloudflare**.

---

## 1. Generar y usar claves SSH seguras

En tu máquina local (por ejemplo, Windows o Linux), genera un par de claves para conectarte sin contraseña.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

Esto crea dos archivos:
- `id_ed25519` → clave privada (no compartir).
- `id_ed25519.pub` → clave pública (para copiar al servidor).

### Copiar la clave pública a la Raspberry Pi

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub syslogic@<IP_TAILSCALE_RASPBERRY>
```

O manualmente:

```bash
cat ~/.ssh/id_ed25519.pub | ssh syslogic@<IP_TAILSCALE_RASPBERRY> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Con esto, ya puedes ingresar sin contraseña:

```bash
ssh syslogic@<IP_TAILSCALE_RASPBERRY>
```

---

## 2. Configurar permisos SSH y endurecer el servicio

Edita el archivo de configuración del servicio SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

Verifica o agrega las siguientes líneas (sin “#” delante):

```
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
AuthorizedKeysFile .ssh/authorized_keys
```

Guarda con `CTRL+O` y `CTRL+X`.  
Reinicia el servicio SSH:

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

Salida esperada:
```
# ● ssh.service - OpenBSD Secure Shell server
#    Loaded: loaded (/lib/systemd/system/ssh.service; enabled)
#    Active: active (running)
```

---

## 3. Configurar Firewall UFW (Uncomplicated Firewall)

Instala y configura UFW para permitir solo los servicios necesarios.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment "SSH acceso interno Tailscale"
sudo ufw allow from 100.0.0.0/8 comment "Tailscale network"
sudo ufw allow 41641/udp comment "Tailscale service"
sudo ufw enable
sudo ufw status verbose
```

Esto garantiza que solo Tailscale y el acceso local puedan comunicarse con la Raspberry Pi.

---

## 4. Alertas de inicio de sesión SSH

Para recibir correos cada vez que alguien se conecte por SSH, instalamos y configuramos **msmtp**.

```bash
sudo apt install msmtp-mta mailutils -y
```

### Configuración de msmtp

```bash
sudo nano /etc/msmtprc
```

Ejemplo de configuración (usa tus datos de Hostinger, sin exponer contraseñas):

```
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        default
host           smtp.tuservidor.com
port           587
from           notificaciones@tu-dominio.com
user           notificaciones@tu-dominio.com
passwordeval   "cat /etc/.smtp_pass"
```

Guarda la contraseña del correo en un archivo seguro:

```bash
echo "TuContraseñaSMTP" | sudo tee /etc/.smtp_pass >/dev/null
sudo chmod 600 /etc/.smtp_pass
```

Verifica el envío:

```bash
echo "Prueba de correo desde Raspberry Pi" | mail -s "Test SMTP" tu-correo@ejemplo.com
```

### Script de alerta SSH

```bash
sudo nano /etc/profile.d/ssh_alert.sh
```

Contenido:

```bash
#!/usr/bin/env bash
IP=$(echo $SSH_CONNECTION | awk '{print $1}')
DATE=$(date "+%Y-%m-%d %H:%M:%S %Z")
USER=$(whoami)
HOST=$(hostname)
MSG="Nuevo login SSH en $HOST\nDesde: $IP\nFecha: $DATE\nUsuario: $USER"
echo -e "$MSG" | mail -s "SSH Login Alert - $HOST" notificaciones@tu-dominio.com
```

Dar permisos de ejecución:

```bash
sudo chmod +x /etc/profile.d/ssh_alert.sh
```

Reinicia sesión SSH y deberías recibir el correo de alerta.

---

## 5. Verificación de puertos abiertos

Comprueba que solo los servicios necesarios estén escuchando:

```bash
sudo ss -tulpen
```

Deberías ver solo:
- `22/tcp` → SSH  
- `41641/udp` → Tailscale  
- `8080/tcp` → Docker (Nextcloud)

---

## Resultado final

Tu sistema ahora cuenta con:
- Acceso SSH seguro mediante clave pública.  
- Firewall UFW bloqueando conexiones externas.  
- Alertas automáticas por correo en cada inicio de sesión.  
- Integración segura con Tailscale y Cloudflare para acceso remoto cifrado.






