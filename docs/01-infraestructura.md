
# 01 - Infraestructura

## Hardware
- Raspberry Pi 5 — 8GB RAM
- SSD externo (512GB)

## Sistema Operativo
- Ubuntu Server 24.04 LTS (64-bit ARM)

## Red
- ISP con puertos entrantes bloqueados
- Solución: uso de túnel seguro para exponer servicios sin abrir puertos
- Acceso privado por **Tailscale** (VPN mesh)

## DNS
- Dominio gestionado en proveedor externo (ej: Hostinger)
- Proxy activado en Cloudflare para ocultar IP pública

## Acceso remoto
- SSH habilitado solo con claves (no passwords)

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

- Restringido a la interfaz privada de Tailscale
