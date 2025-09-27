# 07 - Errores y Lecciones Aprendidas

## Introducción
En este documento se recopilan los errores encontrados durante la implementación de Nextcloud en Raspberry Pi 5, junto con el diagnóstico y las soluciones aplicadas.  
Esto refleja problemas reales de un entorno de producción y cómo fueron superados.

---

## Índice de errores
1. Nextcloud en Raspberry Pi (incompatibilidad ARM)  
2. Problemas de permisos en carpeta `data`  
3. Puertos bloqueados por ISP  
4. Backups incompletos (sin base de datos)  
5. Notificaciones por correo (`msmtp`)  
6. Contenedor caído sin aviso (sin watchdog)  
7. Logs de Docker sin control de crecimiento  

---

## 1) Nextcloud en Raspberry Pi (incompatibilidad ARM)

**Problema:**  
Al usar imágenes Docker no oficiales, Nextcloud fallaba en Raspberry Pi con `exec format error`.

**Síntomas observados:**  
- Contenedor no arrancaba.  
- Logs de `docker logs nextcloud_app` mostraban errores de arquitectura.  

**Diagnóstico:**  

Verificación de arquitectura con:
```bash
uname -m   # -> aarch64
```
confirmación de imagen incompatible con:

```bash
docker manifest inspect nextcloud
```

Solución aplicada:

-Migrar a imágenes multi-arch:

  - Nextcloud: lscr.io/linuxserver/nextcloud:latest

  - MariaDB: mariadb:10.11

- Verificar compatibilidad antes de desplegar (docker manifest inspect).

-----

## 2) Problemas de permisos en carpeta data

Problema:
Nextcloud no podía escribir en /srv/nextcloud/data.

Síntomas observados:

- Error en el instalador web: "Cannot create data directory".

- Archivos creados con permisos root.

Diagnóstico:
Revisión de permisos:

```bash
ls -ld /srv/nextcloud/data
```
"Mostraba propietario root."

Solución aplicada:
Asignar al usuario correcto (uid=1000 usado por la imagen de LinuxServer.io):

```bash
sudo chown -R 1000:1000 /srv/nextcloud/app /srv/nextcloud/data
docker restart nextcloud_app
```

------

## 3) Puertos bloqueados por ISP

Problema:
No era posible acceder desde Internet al servidor, aunque el router tenía NAT configurado.

Síntomas observados:

- Escaneo desde el exterior (nmap) → puertos cerrados.

- Acceso LAN normal, acceso externo imposible.

Diagnóstico:
Confirmación de que el ISP bloqueaba puertos comunes (80/443).

Solución aplicada:

- Uso de Cloudflare Tunnel para exponer Nextcloud de forma segura.

- SSH remoto mediante Tailscale VPN, independiente del túnel.

-------------

## 4) Backups incompletos (sin base de datos)

Problema:
Los primeros backups incluían solo los archivos de usuario, no la base de datos.

Síntomas observados:

- Restauración → los usuarios y configuraciones no coincidían.

- Carpeta data/ estaba completa pero Nextcloud mostraba errores.

Diagnóstico:
Faltaba la base de datos en el backup (mysqldump).

Solución aplicada:

- Añadir mysqldump al script backup-nextcloud.sh.

- Guardar db.sql junto a data.tgz.


------------

## 5) Notificaciones por correo (msmtp)

Problema:
msmtp fallaba con autenticación o TLS, sin enviar correos.

Síntomas observados:

- Logs de msmtp → "authentication failed" o "TLS handshake error".

- Ningún correo recibido en las pruebas.

Diagnóstico:
Archivo .msmtprc mal configurado (puerto, STARTTLS, credenciales).

Solución aplicada:

- Configurar correctamente .msmtprc con tls on y tls_starttls on.

- Usar contraseña de aplicación (cuando el proveedor lo exige).

- Centralizar envíos con notify.sh.

----------

## 6) Contenedor caído sin aviso (sin watchdog)

Problema:
El contenedor nextcloud_app se detenía sin notificación.

Síntomas observados:

- docker ps no lo mostraba en ejecución.

- Nextcloud inaccesible.

Diagnóstico:
Revisión de logs confirmaba caídas esporádicas de PHP/FPM.

Solución aplicada:

- Crear watch-nextcloud.sh → revisa contenedor cada 10 min.

- Reinicia si está detenido.

----------

## 7) Logs de Docker sin control de crecimiento

Problema:
Los logs en /var/lib/docker/containers crecían sin límite, llenando el disco.

Síntomas observados:

- df -h mostraba partición casi llena.

- Directorio de logs superaba varios GB.

Diagnóstico:
Confirmado con:

```bash
du -sh /var/lib/docker/containers/*
```

Solución aplicada:

Configurar rotación de logs en /etc/docker/daemon.json:


```bash
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Reinicia Docker:
```bhas
sudo systemctl restart docker
```



