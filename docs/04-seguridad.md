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
- 
```bash
sudo ufw allow from 192.xxx.1.0/24 to any port 80,443 proto tcp
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






