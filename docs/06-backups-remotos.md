### Copia off-site al equipo Windows por Tailscale

**Opción A – `scp` (simple, no incremental):**
```bash
# Variables de destino (ajustar)
WIN_USER="usuario_windows"
WIN_HOST="100.x.x.x"                 # IP Tailscale del PC Windows
WIN_DIR="/c/NextcloudBackups"        # Carpeta en Windows

# Crear carpeta destino y subir el último backup
ssh "$WIN_USER@$WIN_HOST" "mkdir -p '$WIN_DIR'"
scp -r "$BACKUP_DIR/$STAMP" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```

**Opción B – `rsync` (incremental, requiere rsync remoto):**
```bash
# Réplica completa de la carpeta de backups
rsync -avh --delete "$BACKUP_DIR/" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```
