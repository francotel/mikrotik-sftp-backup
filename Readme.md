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
:local sftpServer "192.168.100.9"       # IP de tu servidor SFTP
:local sftpPort 2022                    # Puerto SFTP (22 por defecto, o el que uses)
:local sftpUser "mikrotik"              # Usuario SFTP
:local sftpPass "TuContraseñaSegura"    # Contraseña SFTP

# --- 2. Definir la carpeta en el servidor ---
# Escribe EXACTAMENTE el nombre de la carpeta que creaste en el servidor SFTP
:local sftpFolder "Router Oficina 1"    

# --- 3. Obtener el nombre del router automáticamente ---
:local deviceModel [/system identity get name]

# --- 4. Variables de fecha y hora ---
:local fecha [/system clock get date]
:local hora [/system clock get time]

# Nombre del archivo: Identidad del router + fecha + hora
:local nombreBackup ($deviceModel . "-backup-" . $fecha . "-" . $hora)
:local nombreExport ($deviceModel . "-export-" . $fecha . "-" . $hora)

# --- 5. Crear los respaldos localmente ---
/system backup save name=$nombreBackup password="clave-del-backup"
/export file=$nombreExport
:delay 5s

# --- 6. Construir las URLs completas ---
:local urlBackup ("sftp://" . $sftpServer . ":" . $sftpPort . "/" . $sftpFolder . "/" . $nombreBackup . ".backup")
:local urlExport ("sftp://" . $sftpServer . ":" . $sftpPort . "/" . $sftpFolder . "/" . $nombreExport . ".rsc")

# --- 7. Subir archivos ---
/tool fetch url=$urlBackup src-path=($nombreBackup . ".backup") user=$sftpUser password=$sftpPass upload=yes
/tool fetch url=$urlExport src-path=($nombreExport . ".rsc") user=$sftpUser password=$sftpPass upload=yes

# --- 8. Borrar archivos locales ---
/file remove [find name=($nombreBackup . ".backup")]
/file remove [find name=($nombreExport . ".rsc")]
```

### 2. Programar la Tarea (Scheduler)

Para que el respaldo se ejecute automáticamente todos los días, crea una tarea en **System > Scheduler** o ejecuta el siguiente comando en la terminal:

```routeros
/system scheduler add name="backup-diario-sftp" start-time=03:00:00 interval=1d on-event="/system script run backup-sftp"
```

<img width="1122" height="1402" alt="Backups centralizados de MikroTik" src="https://github.com/user-attachments/assets/a600486c-50b5-47c0-954f-30fb6cba8120" />

