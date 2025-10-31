# 01 – Infraestructura

En esta sección se prepara la infraestructura base del proyecto **Nextcloud en Raspberry Pi 5**, incluyendo la instalación del sistema operativo, configuración de red, acceso remoto mediante **SSH**, red privada con **Tailscale**, y la estructura de carpetas inicial del entorno.

---

## Requisitos previos

- Raspberry Pi 5 (8GB recomendado) + fuente oficial.
- microSD (≥32GB) o SSD USB.
- Otro equipo (Windows/macOS/Linux) para grabar la imagen.
- Red local con DHCP habilitado (router normal).

---

## 1) Instalar el sistema operativo en la microSD/SSD

> Recomendado: **Ubuntu Server 24.04 LTS (ARM64)** o **Raspberry Pi OS Lite 64‑bit**.

Descarga oficial:
- Ubuntu: https://ubuntu.com/download/raspberry-pi  
- Raspberry Pi OS: https://www.raspberrypi.com/software/operating-systems/

Graba la imagen con **Raspberry Pi Imager** o **balenaEtcher**. Conecta la tarjeta/SSD a la Raspberry y enciende.

---

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

---

## 5) Actualizar sistema y utilidades básicas

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git jq htop unzip net-tools
sudo reboot
```

Tras reiniciar, valida versión:
```bash
lsb_release -a
```

Ejemplo de salida:
```
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.3 LTS
Release:        24.04
Codename:       noble
```

---

## 6) IP estática (opcional)

### A) Ubuntu (Netplan)
Edita el archivo (interfaz típica `eth0`):
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
Ejemplo **YAML** (ajusta a tu red):
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.1.145/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```
Aplica cambios:
```bash
sudo netplan apply
```

### B) Raspberry Pi OS (dhcpcd)
```bash
sudo nano /etc/dhcpcd.conf
```
Añade al final (ajusta a tu red):
```
interface eth0
static ip_address=192.168.1.145/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1 8.8.8.8
```
Guarda y reinicia:
```bash
sudo reboot
```

---

## 7) Instalar y unir **Tailscale** (acceso privado)

Instala:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Inicia y autentica (abre el enlace que mostrará en tu navegador):
```bash
sudo tailscale up --ssh
```
Comprueba IP privada de Tailscale (100.x.x.x):
```bash
tailscale ip -4
tailscale status
```

> A partir de ahora, podrás acceder por `ssh` usando **la IP de Tailscale** desde tus otros dispositivos con Tailscale.

---

## 8) Preparar **Cloudflare Tunnel** (solo instalación/autenticación)


Instala `cloudflared`:
```bash
curl -fsSL https://pkg.cloudflare.com/install.sh | sudo bash
sudo apt install -y cloudflared
```

Autentica contra tu cuenta de Cloudflare (se abrirá un enlace web):
```bash
cloudflared tunnel login
```

Crea el túnel (guarda el nombre para usarlo en el paso 02):
```bash
cloudflared tunnel create nextcloud-tunnel
```
Comprueba que se ha creado el archivo de credenciales (ruta típica):
```bash
ls -l ~/.cloudflared/
```

> En el paso 02 definiremos las **rutas/ingress** y el subdominio en Cloudflare (CNAME con proxy naranja).

---

## 9) Estructura de carpetas base (para los siguientes pasos)

Crea las carpetas que usaremos más adelante (sin Docker todavía):
```bash
mkdir -p ~/docker/nextcloud/{db,app}
mkdir -p ~/docker/automation/{scripts,logs,backups}
```

---

## 10) Comprobaciones rápidas

```bash
hostnamectl
ip a | grep inet
tailscale status
```
