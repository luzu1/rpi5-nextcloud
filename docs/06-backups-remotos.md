# 06 – Backups remotos (Windows vía Tailscale + OpenSSH)

Copiaremos automáticamente los backups locales de la Raspberry Pi hacia un **PC Windows** usando **Tailscale** (red privada) y **OpenSSH Server** (en Windows).  
Este paso parte de que ya tenés:
- Backups locales creados por el Paso 04 (en `~/docker/automation/backups`).
- Tailscale funcionando (Paso 01).
- msmtp configurado (Paso 03) – *opcional, solo para notificaciones*.

---

## 1) Requisitos en Windows (instalar y habilitar OpenSSH Server)

> Ejecutar en **Windows 10/11** (PowerShell *como Administrador*).

**Instalar el servicio:**
```powershell
# Ver si está instalado
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'

# Instalar si falta
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Habilitar y arrancar
Set-Service -Name sshd -StartupType Automatic
Start-Service sshd
```

**Permitir SSH en el firewall (puerto 22/TCP):**
```powershell
# Crear regla entrante si no existe
New-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -DisplayName "OpenSSH SSH Server (TCP-In)" `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

**Comprobar estado del servicio y escucha:**
```powershell
Get-Service sshd
netstat -abno | Select-String ":22"
```

> Debe verse `sshd` en ejecución y **LISTENING** en el puerto 22.

---

## 2) Crear carpeta de destino para los backups

> Ejecutar en **Windows** (PowerShell, puede ser normal).

```powershell
New-Item -ItemType Directory -Force "C:\NextcloudBackups" | Out-Null
```

---

## 3) Configurar autenticación por clave SSH (sin password)

### 3.1 Generar (o reutilizar) clave en la Raspberry

> Ejecutar en **Raspberry Pi** (usuario normal, *no root*).

```bash
# Si ya tenés una clave, podés saltar este paso
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
# Ver la clave pública
cat ~/.ssh/id_ed25519.pub
```

Copiá el **contenido completo** que imprime (empieza con `ssh-ed25519 AAAA...`).

### 3.2 Cargar la clave en Windows (método GUI rápido)
- Crear la carpeta `C:\Users\<tu_usuario>\.ssh\` si no existe.
- Crear el archivo `authorized_keys` (sin extensión) dentro de esa carpeta.
- Pegar adentro **una línea** con tu clave pública.

> Si Windows lo guarda como `.txt`, renombrá a `authorized_keys` **sin** `.txt`.

### 3.3 Fijar permisos correctos en Windows (evita “Access denied”)

> Ejecutar en **Windows** (PowerShell **como Administrador**).

```powershell
$sshdir = "$env:USERPROFILE\.ssh"
$auth   = "$sshdirauthorized_keys"

# Asegurar carpeta/archivo
New-Item -ItemType Directory -Force $sshdir | Out-Null
if (-not (Test-Path $auth)) { New-Item -ItemType File -Force $auth | Out-Null }

# Tomar propiedad y permisos mínimos (solo tu usuario)
takeown /F $sshdir /R /A | Out-Null
icacls  $sshdir /inheritance:r | Out-Null
icacls  $sshdir /grant "$env:USERNAME:(OI)(CI)F" | Out-Null

takeown /F $auth /A | Out-Null
icacls  $auth /inheritance:r | Out-Null
icacls  $auth /grant "$env:USERNAME:F" | Out-Null
```

> Si sigue marcando “Acceso denegado”, asegurate de abrir PowerShell **como Administrador**.

### 3.4 Endurecer `sshd_config` en Windows (opcional pero recomendado)
> Archivo: `C:\ProgramData\ssh\sshd_config`

- Asegurate de tener:
```
PubkeyAuthentication yes
AuthorizedKeysFile    .ssh/authorized_keys
# Opcional para forzar solo clave (después de probar):
# PasswordAuthentication no
```
> Reiniciar el servicio:
```powershell
Restart-Service sshd
```

---

## 4) Obtener la IP de Tailscale de Windows

> Ejecutar en **Windows** (PowerShell o CMD):
```powershell
ipconfig
```
Buscá el adaptador **Tailscale**, por ejemplo:  
`Dirección IPv4: 100.77.159.93`

> Alternativa en Raspberry (si Windows tiene Tailscale):  
`tailscale status` – ver la IP `100.x.y.z` del nodo Windows.

---

## 5) Probar acceso desde la Raspberry (sin password)

> Ejecutar en **Raspberry Pi**:

```bash
# Reemplazar usuario y la IP 100.x.y.z por la de Windows
ssh -i ~/.ssh/id_ed25519 <usuario_windows>@100.x.y.z "echo OK desde Raspberry"
```

Si está todo correcto, debe imprimir `OK desde Raspberry` **sin pedir contraseña**.

---

## 6) Copia de backups: métodos soportados

> Por defecto usaremos **SCP** (simple, sin incremental).  
> `rsync` solo funcionará si en Windows tenés un `rsync` instalado (MSYS2, cwRsync, WSL, etc.).

### 6.1 SCP (recomendado por simplicidad)

> Ejecutar en **Raspberry**. Copia el último backup generado.

```bash
# Variables de ejemplo (ajustar)
WIN_USER="<usuario_windows>"
WIN_HOST="100.x.y.z"             # IP Tailscale de Windows
WIN_DIR="/c/NextcloudBackups"    # Ruta estilo POSIX aceptada por OpenSSH de Windows

# Crear carpeta destino (idempotente)
ssh "$WIN_USER@$WIN_HOST" "mkdir -p '$WIN_DIR'"

# Subir el directorio del último backup (reemplazá con la carpeta concreta)
STAMP_LATEST="$(ls -1t ~/docker/automation/backups | head -n1)"
scp -r "$HOME/docker/automation/backups/$STAMP_LATEST" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```

> También podés copiar **todos** los backups con:
```bash
scp -r "$HOME/docker/automation/backups" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```

### 6.2 rsync (opcional, si tenés rsync en Windows)

```bash
# Réplica completa e incremental
rsync -avh --delete "$HOME/docker/automation/backups/" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
```

> Si no hay `rsync` del lado Windows, este método fallará. Usá SCP.

---

## 7) Script de ayuda para copias off‑site

Podés crear un script separado para lanzar la copia cuando quieras (por ejemplo tras el backup semanal).

> Archivo en **Raspberry**: `~/docker/automation/scripts/offsite_windows.sh`
```bash
nano ~/docker/automation/scripts/offsite_windows.sh
```

Contenido:
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# === Ajustar según tu entorno ===
WIN_USER="<usuario_windows>"
WIN_HOST="100.x.y.z"
WIN_DIR="/c/NextcloudBackups"

# Carpeta local de backups (Paso 04)
SRC_DIR="$HOME/docker/automation/backups"

# Asegurar carpeta remota
ssh "$WIN_USER@$WIN_HOST" "mkdir -p '$WIN_DIR'" || exit 1

# Copia con SCP (simple). Descomenta rsync si lo tenés disponible en Windows.
scp -r "$SRC_DIR" "$WIN_USER@$WIN_HOST:$WIN_DIR/"

# rsync -avh --delete "$SRC_DIR/" "$WIN_USER@$WIN_HOST:$WIN_DIR/"
echo "Copia off-site a Windows completada."
```

Permisos y prueba:
```bash
chmod +x ~/docker/automation/scripts/offsite_windows.sh
bash ~/docker/automation/scripts/offsite_windows.sh
```

> **Integración con el backup semanal:**  
> Al final de `backup_nextcloud.sh` (Paso 04), podés **invocar** este script para que, tras crear el backup local, se envíe a Windows.  
> Esto mantiene la lógica separada y facilita el mantenimiento.

---

## 8) Comprobación en Windows

Verificá que en `C:\NextcloudBackups\` aparezcan las carpetas tipo `AAAA-MM-DD_HH-MM` con:
- `nextcloud_html.tar.zst`
- `db.sql.zst`
- `SHA256SUMS`

Podés validar integridad localmente (en Raspberry):
```bash
cd ~/docker/automation/backups/AAAA-MM-DD_HH-MM
sha256sum -c SHA256SUMS
```

---

## 9) Solución de problemas (FAQ)

- **Me pide contraseña al hacer SSH**  
  - Verificar que la clave pública esté en `C:\Users\<usuario>\.ssh\authorized_keys` (una sola línea).  
  - Comprobar `sshd_config` (Windows): `PubkeyAuthentication yes` y `AuthorizedKeysFile .ssh/authorized_keys`.  
  - Reiniciar `sshd`: `Restart-Service sshd` (PowerShell Admin).  
  - Permisos con `icacls` como se indicó arriba.

- **Permiso denegado al escribir `authorized_keys`**  
  - Usar PowerShell **como Administrador**. Repetir los comandos `takeown` / `icacls` del paso 3.3.

- **`rsync` falla**  
  - En Windows no hay `rsync` por defecto. Usar **SCP** o instalar `rsync` (MSYS2/WSL).

- **No conecta a la IP 100.x.y.z**  
  - Verificar que Tailscale está conectado en ambos equipos.  
  - Comprobar firewall de Windows (regla creada) y que `sshd` está **Running**.

---

Con esto dejás implementado un **flujo off‑site** hacia tu equipo Windows, manteniendo los backups **fuera de la Raspberry** y dentro de tu red privada (Tailscale).  
La restauración se hace de la misma forma que en el Paso 04, usando los archivos copiados a Windows.
