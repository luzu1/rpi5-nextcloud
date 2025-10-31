# 01 - Infraestructura

## Introducción

En esta sección se prepara toda la infraestructura base del proyecto **Nextcloud en Raspberry Pi 5**, incluyendo el sistema operativo, red privada mediante **Tailscale**, acceso público seguro con **Cloudflare Tunnel** y la gestión de DNS mediante **Hostinger**.  
El objetivo es dejar el entorno completamente funcional para los siguientes pasos de instalación de Docker, seguridad y automatización.

---

## 1) Instalación base del sistema

Se recomienda utilizar **Ubuntu Server 24.04 LTS (aarch64)** o **Raspberry Pi OS Lite 64-bit**.

Descarga la imagen oficial desde:  
🔗 [https://ubuntu.com/download/raspberry-pi](https://ubuntu.com/download/raspberry-pi)

Graba la imagen en una microSD o SSD con **Raspberry Pi Imager**.  
Luego conecta la Raspberry Pi a la red local por cable y al iniciar, habilita SSH desde un monitor o desde Imager antes de grabar la imagen.

## 2) Habilitar SSH (según sistema)

### A) Ubuntu Server 24.04 (ARM64)
SSH viene habilitado. Usuario inicial: **ubuntu** (te pedirá cambiar contraseña al primer login).

### B) Raspberry Pi OS Lite (64‑bit)
Crea un archivo vacío llamado **`ssh`** en la partición **boot** antes del primer arranque.

En Windows (después de grabar la imagen, aparecerá la unidad `boot`):
```bash
echo. > X:\ssh
```
> Sustituye `X:` por la letra de unidad de la partición **boot**.

---

## 3) Encontrar la IP de la Raspberry

El método más simple es **mirar la lista de clientes** en tu router. Alternativas:

- Con monitor/teclado en la Raspberry:  
  ```bash
  ip a | grep inet
  ```
- Desde otro equipo en la LAN (ejemplo, red 192.168.1.0/24):  
  ```bash
  ping -n 1 raspberrypi.local   # (Windows, si mDNS)
  arp -a                        # tabla ARP
  ```

Ejemplo de salida (en la Raspberry):

>inet 192.168.1.145/24 brd 192.168.1.255 scope global eth0


---

## 4) Primer acceso por SSH y cambio de contraseña

Conéctate desde tu PC (sustituye `IP_RASPBERRY`):

**Ubuntu Server**:
```bash
ssh ubuntu@IP_RASPBERRY
# contraseña inicial: ubuntu  (obliga a cambiarla)
```

**Raspberry Pi OS**:
```bash
ssh pi@IP_RASPBERRY
# cambia la contraseña con:
passwd
```

Define un **hostname** identificable:
```bash
sudo hostnamectl set-hostname raspi-nextcloud
```

 Al iniciar por primera vez, el sistema solicitará cambiar la contraseña.

## Actualiza paquetes y herramientas básicas:
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git ufw jq htop net-tools unzip
sudo reboot
```

---

## 5) Configuración de red y hostname

Verifica la IP asignada:
```bash
hostname -I
```

Ejemplo de salida:
```
192.168.1.145
```

Asigna un nombre único al dispositivo:
```bash
sudo hostnamectl set-hostname raspberry
```

### (Opcional) IP estática
Edita el archivo de configuración de Netplan:
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Ejemplo:
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.145/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

Aplica los cambios:
```bash
sudo netplan apply
```

---

## 6) Instalación de Tailscale (red privada)

Tailscale crea una red privada cifrada entre tus dispositivos (sin abrir puertos).  
Instálalo y autentícalo con tu cuenta.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Verifica que esté conectado correctamente:
```bash
tailscale status
```

Ejemplo de salida:
```
100.77.159.93   raspberry   user@   linux   active; relay "fra"
```

---

## 7) Configuración de Cloudflare Tunnel (acceso público)

Cloudflare Tunnel permite exponer servicios como Nextcloud a Internet sin abrir puertos en el router.  
Instala el cliente oficial de Cloudflare en la Raspberry Pi:

```bash
sudo mkdir -p /srv/cloudflared
cd /srv/cloudflared
sudo curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -o cloudflared
sudo chmod +x cloudflared
sudo mv cloudflared /usr/local/bin/
```

Autentica tu túnel con tu cuenta de Cloudflare:
```bash
cloudflared tunnel login
```

Crea el túnel:
```bash
cloudflared tunnel create nextcloud-tunnel
```

Agrega la configuración base:
```bash
sudo nano /etc/cloudflared/config.yml
```

Ejemplo de configuración:
```yaml
tunnel: nextcloud-tunnel
credentials-file: /root/.cloudflared/<UUID>.json

ingress:
  - hostname: nextcloud.tudominio.com
    service: http://localhost:8080
  - service: http_status:404
```

Inicia y habilita el servicio:
```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

---

## 8) Configuración DNS en Hostinger

1. Entra al panel de **Hostinger**.  
2. Abre la sección **Zona DNS** de tu dominio.  
3. Crea un registro **CNAME** apuntando al dominio gestionado por Cloudflare.

Ejemplo:
```
Tipo: CNAME
Nombre: nextcloud
Valor: nextcloud.tunnel.cloudflare.com
TTL: 3600
```

Asegúrate de tener activado el proxy (ícono de nube naranja) en Cloudflare para proteger la IP de la Raspberry Pi.

---

## 9) Verificación final

Comprueba la conectividad entre todos los servicios:

```bash
tailscale status
ping 1.1.1.1 -c 4
cloudflared tunnel list
```

Deberías ver tu túnel activo y accesible mediante el dominio configurado en Hostinger.

---
