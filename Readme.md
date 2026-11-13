# 🚀 Respaldo Automatizado de MikroTik vía SFTP

Este repositorio contiene los scripts y la documentación necesaria para automatizar el respaldo de routers MikroTik hacia cualquier servidor SFTP (TrueNAS, Linux, Synology, etc.). 

El objetivo es centralizar los respaldos de red de forma segura, organizada y sin intervención manual.

## 📋 Características

*   **Seguridad:** Utiliza el protocolo SFTP (SSH File Transfer Protocol) en lugar de FTP tradicional.
*   **Organización:** Cada router guarda sus archivos en una carpeta específica que el administrador define previamente en el servidor.
*   **Automatización:** Los archivos se nombran automáticamente usando la identidad del router y la fecha/hora del sistema, evitando sobreescribir respaldos antiguos.
*   **Limpieza:** El script elimina los archivos locales del router después de una subida exitosa, ahorrando espacio en la memoria flash del equipo.

## 🛠️ Requisitos Previos

1.  **Servidor SFTP:** Debes tener un servidor SFTP configurado y accesible desde la red de tus routers (puede ser un TrueNAS con SFTPGo, un servidor Linux con OpenSSH, etc.).
2.  **Usuario y Credenciales:** Un usuario y contraseña con permisos de escritura en el directorio de respaldos.
3.  **Routers MikroTik:** Con RouterOS v6 o v7 (se recomienda v7+), con conectividad hacia el servidor SFTP.
4.  **Carpetas Creadas:** ⚠️ **IMPORTANTE:** Debes crear manualmente las carpetas en el servidor SFTP para cada router (ej. `Router Oficina 1`, `Router Sucursal`). El script **no** crea carpetas en el servidor de forma automática.

<img width="960" height="596" alt="image" src="https://github.com/user-attachments/assets/08cc7fc7-2c4d-4b60-ba3f-898e8b4a1997" />

## ⚙️ Configuración en el Router MikroTik

Sigue estos pasos para configurar cada router:

### 1. Crear el Script

Ve a **System > Scripts** y crea un nuevo script llamado `backup-sftp`. Pega el siguiente código y ajusta las variables de la sección 1 y 2 con tus datos reales:

```routeros
# --- 1. Variables de configuración del Servidor SFTP ---
:local sftpServer "192.168.100.X"
:local sftpPort 2022
:local sftpUser "user"
:local sftpPass "password"

# --- 2. Definir la carpeta en el servidor ---
# ⚠️ CAMBIA ESTO en cada router por el nombre exacto de su carpeta en SFTPGo
:local sftpFolder "RB4011-ADMIN-1"    

# --- 3. Obtener el nombre del router automáticamente ---
:local deviceModel [/system identity get name]

# --- 4. Variables de fecha y hora (Compatible con v6 y v7) ---
# Extraemos día, mes y año para evitar las "/"
:local dateRaw [/system clock get date]
:local day [:pick $dateRaw 4 6]
:local month [:pick $dateRaw 0 3]
:local year [:pick $dateRaw 7 11]

# Extraemos la hora y evitamos los ":"
:local timeRaw [/system clock get time]
:local hour [:pick $timeRaw 0 2]
:local min [:pick $timeRaw 3 5]
:local sec [:pick $timeRaw 6 8]

# Construimos un nombre limpio: Router-backup-2026-nov-13-14-30-00
:local nombreBase ($deviceModel . "-backup-" . $year . "-" . $month . "-" . $day . "-" . $hour . "-" . $min . "-" . $sec)
:local nombreExport ($deviceModel . "-export-" . $year . "-" . $month . "-" . $day . "-" . $hour . "-" . $min . "-" . $sec)

# --- 5. Crear los respaldos localmente (Sin contraseña) ---
/system backup save name=$nombreBase
/export file=$nombreExport
:delay 5s

# --- 6. Construir las URLs completas ---
:local urlBackup ("sftp://" . $sftpServer . ":" . $sftpPort . "/" . $sftpFolder . "/" . $nombreBase . ".backup")
:local urlExport ("sftp://" . $sftpServer . ":" . $sftpPort . "/" . $sftpFolder . "/" . $nombreExport . ".rsc")

# --- 7. Subir archivos ---
/tool fetch url=$urlBackup src-path=($nombreBase . ".backup") user=$sftpUser password=$sftpPass upload=yes
/tool fetch url=$urlExport src-path=($nombreExport . ".rsc") user=$sftpUser password=$sftpPass upload=yes

# --- 8. Borrar archivos locales para no llenar la memoria ---
/file remove [find name=($nombreBase . ".backup")]
/file remove [find name=($nombreExport . ".rsc")]
```

### 2. Programar la Tarea (Scheduler)

Para que el respaldo se ejecute automáticamente todos los días, crea una tarea en **System > Scheduler** o ejecuta el siguiente comando en la terminal:

```routeros
/system scheduler add name="backup-diario-sftp" start-time=03:00:00 interval=1d on-event="/system script run backup-sftp"
```

<img width="1122" height="1402" alt="Backups centralizados de MikroTik" src="https://github.com/user-attachments/assets/a600486c-50b5-47c0-954f-30fb6cba8120" />

