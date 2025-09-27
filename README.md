# Nextcloud en Raspberry Pi 5 con Docker

**Objetivo:** Montar un servidor Nextcloud seguro y automatizado sobre Raspberry Pi 5 + Docker, con acceso privado vía Tailscale y público controlado con Cloudflare Tunnel.

---

## 📂 Documentación

- [00 - Overview](docs/00-overview.md)
- [01 - Infraestructura](docs/01-infraestructura.md)
- [02 - Docker & Nextcloud](docs/02-docker&nextcloud.md)
- [03 - Seguridad](docs/03-seguridad.md)
- [04 - Automatización](docs/04-automatizacion.md)
- [05 - Vaultwarden](docs/05-vaultwarden.md)
- [Errores y lecciones aprendidas](docs/errores.md)

---

## ⚙️ Componentes principales
- Raspberry Pi 5 (8GB RAM, 512GB SSD)
- Ubuntu Server 24.04 (aarch64)
- Docker + Docker Compose
- Nextcloud + MariaDB
- Cloudflare Tunnel + Tailscale
- Scripts de automatización y backups

---

## 🔒 Seguridad
- SSH solo con claves vía interfaz `tailscale0`.
- Firewall UFW bloqueando todo salvo LAN/Tailscale.
- Cloudflare Tunnel para acceso externo sin abrir puertos.

---

## 📜 Licencia
MIT — ver [LICENSE](LICENSE).
