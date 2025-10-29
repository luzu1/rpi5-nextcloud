## Introducción
[Vaultwarden](https://github.com/dani-garcia/vaultwarden) es una implementación ligera y compatible de Bitwarden, ideal para montar un gestor de contraseñas auto-hospedado.  
En este proyecto se instala en la Raspberry Pi 5, usando **Docker y Docker Compose**, pero en un stack **separado del de Nextcloud**.

---

## 1) Preparar estructura de directorios

Creamos carpetas para datos y configuración de Vaultwarden:

```bash
sudo mkdir -p /srv/vaultwarden/{data}
sudo chown -R 1000:1000 /srv/vaultwarden
```

-------------

## 2) Variables de entorno

Creamos ~/vaultwarden/.env (privado, no subir a GitHub).
En el repo subiremos env.example.

```bash
# Zona horaria
TZ=Europe/Madrid

# Credenciales admin (interfaz de administración de Vaultwarden)
ADMIN_TOKEN=CAMBIA_ESTE_VALOR

# Configuración básica
SIGNUPS_ALLOWED=true
```

--------------

## 3) Archivo docker-compose.yml

```bash
version: "3.8"

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - TZ=${TZ}
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - SIGNUPS_ALLOWED=${SIGNUPS_ALLOWED}
    volumes:
      - /srv/vaultwarden/data:/data
    ports:
      - "8081:80"   # acceso LAN en puerto 8081
```

-----

## 4) Levantar el stack

```bash
cd ~/vaultwarden
docker compose pull
docker compose up -d
docker compose ps
```

Verificar logs:

```bash
docker logs -f vaultwarden
```

-----------

## 5) Acceso inicial

Interfaz web Vaultwarden:

- http://<IP-LAN>:8081

Interfaz de administración:

- http://<IP-LAN>:8081/admin
(requiere ADMIN_TOKEN configurado en .env).

--------

6) Integración segura

-No exponer el puerto 8081 directamente a Internet.

-Usar Cloudflare Tunnel o VPN (Tailscale) para acceso externo seguro.

-Habilitar HTTPS en el proxy si se publica públicamente.

-------

## 7) Buenas prácticas

Mantener .env fuera del repo, subir solo .env.example.

Respaldar regularmente la carpeta /srv/vaultwarden/data.

Deshabilitar SIGNUPS_ALLOWED después de registrar usuarios iniciales.

Usar 2FA en la cuenta de administración.


   
