# Nextcloud en Raspberry Pi 5 — Docker · MariaDB · Cloudflare Tunnel · Tailscale

Infraestructura personal segura con **Nextcloud** y **MariaDB** en **Raspberry Pi 5** (ARM64) usando **Docker**. Acceso externo mediante **Cloudflare Tunnel**; acceso de administración por **Tailscale (SSH)**. Backups automatizados y alertas básicas.

- OS: Ubuntu Server 24.04 (aarch64)
- Hardware: RPi 5 · 8GB RAM · SSD 512GB
- ISP: Digi (puertos bloqueados)
- DNS: Hostinger (`zonasecure.xyz`)
- Dominio: `cloud.zonasecure.xyz`

## Documentación
- [00 - Overview](docs/00-overview.md)
- [01 - Infraestructura](docs/01-infraestructura.md)
- [02 - Docker & Contenedores](docs/02-docker-contenedores.md)
- [03 - Backups](docs/03-backups.md)
- [04 - Seguridad](docs/04-seguridad.md)
- [05 - Automatización](docs/05-automatizacion.md)
- [06 - Monitoreo](docs/06-monitoreo.md)
- [07 - Errores & Lecciones](docs/07-errores.md)

## Despliegue rápido (resumen)
1. Copia `.env.example` a `.env` y completa variables.
2. Arranca servicios:
   ```bash
   docker compose up -d
