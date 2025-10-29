# **Nextcloud Raspberry Pi 5 (Docker + Tailscale + Cloudflare)**

**Objetivo:**  

Implementar un servidor Nextcloud completamente seguro, automatizado y autosuficiente sobre una Raspberry Pi 5.  
El sistema integra red privada mediante Tailscale, acceso público controlado por Cloudflare Tunnel, automatización de tareas, alertas por correo y copias de seguridad remotas.

---

## **Documentación**

- [00 - Overview](docs/00-overview.md) : Resumen general del proyecto, alcance, objetivos y componentes principales.

- [01 - Infraestructura](docs/01-infraestructura.md) : Preparación del entorno: Raspberry Pi OS, red, Tailscale, Cloudflare Tunnel y estructura de carpetas base.

- [02 - Docker & Nextcloud](docs/02-docker-nextcloud.md) : Instalación de Docker y Docker Compose, configuración del contenedor de Nextcloud y MariaDB, variables .env y comprobaciones de estado.

- [03 - Seguridad](docs/03-seguridad.md) :  Endurecimiento del sistema con UFW, SSH seguro, Tailscale, Cloudflare Tunnel y buenas prácticas de acceso.

- [04 - Automatización](docs/04-automatizacion.md) : Scripts de mantenimiento automático para watchdog, actualizaciones de sistema, contenedores y verificación de servicios.

- [05 - Alertas SSH & Correo](docs/05-alertas-ssh-correo.md) : Configuración de notificaciones por correo (msmtp), alertas de inicio de sesión SSH y estado de Nextcloud.

- [06 - Backups Remotos (Windows)](docs/06-backups-remotos.md) : Automatización de copias de seguridad locales y transferencia a Windows mediante rsync vía Tailscale.

- [07 - Restauración y Mantenimiento](docs/07-restauracion-mantenimiento.md) : Procedimiento para restaurar Nextcloud desde copias de seguridad, comprobaciones finales y buenas prácticas de mantenimiento.

---

---

##  Licencia
MIT — ver [LICENSE](LICENSE).  
