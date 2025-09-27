# Nextcloud en Raspberry Pi 5 con Docker

**Objetivo:** Montar un servidor Nextcloud seguro y automatizado sobre Raspberry Pi 5 con Docker, con acceso privado vía Tailscale y público controlado con Cloudflare Tunnel.  

---

## 📑 Documentación

- [00 - Overview](docs/00-overview.md)  
- [01 - Infraestructura](docs/01-infraestructura.md)  
- [02 - Docker & Nextcloud](docs/02-docker.md)  
- [03 - Seguridad](docs/04-seguridad.md)  
- [04 - Automatización](docs/05-automatizacion.md)  
- [05 - Vaultwarden](docs/05-vaultwarden.md)  
- [07 - Errores y lecciones aprendidas](docs/07-errores.md)  

---

## 🖥️ Infraestructura
[Ver documentación completa](docs/01-infraestructura.md)  
Raspberry Pi 5 (8GB RAM, 512GB SSD) con Ubuntu Server 24.04 (aarch64).  
Conectividad mediante ISP con puertos bloqueados → uso de Tailscale para acceso privado y Cloudflare Tunnel para acceso público seguro.  

---

## 📦 Docker & Nextcloud
[Ver documentación completa](docs/02-docker.md)  
Instalación de Docker y Docker Compose.  
Despliegue de Nextcloud + MariaDB en contenedores ARM64.  
Estructura de carpetas `/srv/nextcloud` para separar datos, configuración y base de datos.  

---

## 🔐 Seguridad
[Ver documentación completa](docs/04-seguridad.md)  
- SSH solo con claves, restringido a interfaz `tailscale0`.  
- Firewall UFW bloqueando todo salvo LAN y Tailscale.  
- Acceso externo gestionado con Cloudflare Tunnel, sin abrir puertos en el router.  

---

## ⚙️ Automatización
[Ver documentación completa](docs/05-automatizacion.md)  
Scripts en `~/automation` para:  
- Backups automáticos (archivos + base de datos).  
- Restauración validada.  
- Watchdog que reinicia contenedores caídos.  
- Actualizaciones periódicas de Docker y sistema operativo.  
- Notificaciones por correo con `notify.sh`.  

---

## 🔑 Vaultwarden
[Ver documentación completa](docs/05-vaultwarden.md)  
Implementación de **Vaultwarden** en un stack de Docker separado.  
Gestor de contraseñas auto-hospedado, accesible en LAN y protegido con túnel seguro para acceso externo.  

---

## 🛠️ Errores y Lecciones Aprendidas
[Ver documentación completa](docs/07-errores.md)  
Problemas reales encontrados y solucionados:  
- Incompatibilidad de imágenes Docker en ARM64.  
- Permisos de carpetas `data`.  
- Puertos bloqueados por ISP → uso de túneles.  
- Backups incompletos → integración de base de datos.  
- Notificaciones con `msmtp` y `notify.sh`.  
- Logs de Docker creciendo sin límite → rotación configurada.  

---

## 📜 Licencia
MIT — ver [LICENSE](LICENSE).  
