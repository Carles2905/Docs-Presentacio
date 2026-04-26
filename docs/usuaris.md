# 👤 Gestión de Roles y Usuarios

En esta sección se documenta la creación y configuración de todos los usuarios del sistema, aplicando el **Principio de Mínimo Privilegio** (*Principle of Least Privilege*): cada usuario dispone únicamente de los permisos imprescindibles para realizar sus funciones y nada más.

---

## 🔐 Principio de Mínimo Privilegio

El principio de mínimo privilegio establece que **ningún usuario, proceso o servicio debe tener más permisos de los estrictamente necesarios** para su función. Esto reduce la superficie de ataque del sistema: si una cuenta es comprometida, el daño posible queda limitado a los recursos a los que ese usuario tenía acceso.

---

## 👑 Usuario: `root` — Adrián (Scrum Master)

### Descripción

El usuario `root` es el superusuario del sistema Linux con **acceso ilimitado** a todos los ficheros, servicios y configuraciones. En nuestro proyecto, este rol corresponde a **Adrián**, que actúa como Scrum Master y responsable técnico global.

### Permisos

- ✅ Acceso total a todos los directorios del sistema
- ✅ Capacidad para instalar y desinstalar paquetes
- ✅ Gestión de servicios con `systemctl`
- ✅ Creación y eliminación de usuarios
- ✅ Modificación de cualquier fichero de configuración

### Consideraciones de Seguridad

!!! danger "Uso responsable del root"
    El acceso como `root` debe usarse **únicamente para tareas que lo requieran explícitamente**. Para tareas cotidianas se debería usar `sudo` desde una cuenta personal con privilegios limitados.

El acceso SSH directo como `root` está **desactivado** en la configuración de seguridad (ver sección [Seguridad y Hardening](seguretat.md)).

---

## 🌐 Usuario: `admin-web` — Carles (Administrador Web)

### Descripción

El usuario `admin-web` es el responsable del servidor web Apache. **Carles** gestiona los ficheros de la aplicación web, los *virtual hosts* y el contenido público del servidor, sin tener acceso a la base de datos ni a los logs del sistema.

### Creación del usuario

```bash
sudo adduser admin-web
```

Durante la creación, se asignará una contraseña y se rellenarán los datos opcionales (nombre completo, etc.).

### Asignación de propiedad del directorio web

El usuario `admin-web` debe ser el **propietario** del directorio `/var/www/html` para poder gestionar los ficheros de la web sin necesitar permisos de `root`:

```bash
sudo chown -R admin-web:admin-web /var/www/html
```

Ajustamos los permisos del directorio para que sean correctos:

```bash
sudo chmod -R 755 /var/www/html
```

Verificamos que la propiedad se ha asignado correctamente:

```bash
ls -la /var/www/html
```

### Permisos

| Recurso | Permiso |
|--------|---------|
| `/var/www/html` | ✅ Lectura y escritura (propietario) |
| `/etc/apache2` | ❌ Sin acceso |
| `/var/log` | ❌ Sin acceso |
| MariaDB | ❌ Sin acceso |
| `sudo` | ❌ No tiene privilegios sudo |

### Cambio de identidad para operar

Para trabajar como `admin-web`, Adrián (root) puede hacer:

```bash
sudo su - admin-web
```

---

## 🗄️ Usuario: `db-backup` — Michael (Seguridad y Backups)

### Descripción

El usuario `db-backup` tiene como única función la realización de **copias de seguridad de las bases de datos** de MariaDB. **Michael** es el responsable de configurar y ejecutar los scripts de `mysqldump`. No tiene acceso al sistema de ficheros web ni a los logs del sistema.

### Creación del usuario del sistema

```bash
sudo adduser --system --no-create-home --shell /bin/bash db-backup
```

La opción `--system` crea un usuario del sistema sin directorio personal, adecuado para tareas automatizadas.

### Creación del usuario en MariaDB

Creamos un usuario en MariaDB con permisos de lectura para la realización de copias:

```bash
sudo mysql -u root -p
```

Una vez dentro del prompt de MariaDB:

```sql
CREATE USER 'db-backup'@'localhost' IDENTIFIED BY 'BackupPass_2024!';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'db-backup'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Script de copia de seguridad

Creamos el directorio donde se guardarán las copias:

```bash
sudo mkdir -p /var/backups/mysql
sudo chown db-backup:db-backup /var/backups/mysql
```

Creamos el script de copia automática:

```bash
sudo nano /usr/local/bin/backup-db.sh
```

Contenido del script:

```bash
#!/bin/bash
FECHA=$(date +%Y-%m-%d_%H-%M-%S)
DEST="/var/backups/mysql"
mysqldump -u db-backup -pBackupPass_2024! --all-databases > "$DEST/backup_$FECHA.sql"
echo "Copia realizada: backup_$FECHA.sql"
```

Damos permisos de ejecución al script:

```bash
sudo chmod +x /usr/local/bin/backup-db.sh
sudo chown db-backup:db-backup /usr/local/bin/backup-db.sh
```

### Permisos

| Recurso | Permiso |
|--------|---------|
| `/var/backups/mysql` | ✅ Lectura y escritura (propietario) |
| MariaDB (lectura) | ✅ `SELECT`, `LOCK TABLES` sobre todas las bases de datos |
| `/var/www/html` | ❌ Sin acceso |
| `/var/log` | ❌ Sin acceso |
| `sudo` | ❌ No tiene privilegios sudo |

---

## 📊 Usuario: `sys-monitor` — Javi (Auditor del Sistema)

### Descripción

El usuario `sys-monitor` es el encargado de la **auditoría y monitorización del servidor**. **Javi** revisa los ficheros de log del sistema para detectar errores, accesos no autorizados o comportamientos anómalos. Tiene permisos de **solo lectura** y no puede modificar ninguna configuración.

### Creación del usuario

```bash
sudo adduser sys-monitor
```

### Configuración de permisos de lectura en los logs

Añadimos `sys-monitor` al grupo `adm`, que en Ubuntu/Debian tiene acceso de lectura a los logs del sistema:

```bash
sudo usermod -aG adm sys-monitor
```

Verificamos que el usuario pertenece al grupo `adm`:

```bash
groups sys-monitor
```

### Verificación de acceso a los logs

El usuario `sys-monitor` podrá leer ficheros como:

```bash
# Logs de autenticación (SSH, sudo, etc.)
tail -n 100 /var/log/auth.log

# Logs generales del sistema
tail -n 100 /var/log/syslog

# Logs de Apache
tail -n 100 /var/log/apache2/access.log
tail -n 100 /var/log/apache2/error.log
```

### Restricciones adicionales

Restricción del acceso al directorio `/var/www/html`:

```bash
sudo chmod o-rx /var/www/html
```

### Permisos

| Recurso | Permiso |
|--------|---------|
| `/var/log` | ✅ Lectura (miembro del grupo `adm`) |
| `/var/log/apache2` | ✅ Lectura |
| `/var/www/html` | ❌ Sin acceso |
| MariaDB | ❌ Sin acceso |
| `sudo` | ❌ No tiene privilegios sudo |

---

## 📋 Resumen de Todos los Usuarios

| Usuario | Miembro del equipo | Rol | Directorio principal | Acceso sudo | Acceso MariaDB |
|--------|-------------------|-----|---------------------|:-----------:|:--------------:|
| `root` | Adrián | Scrum Master / Admin total | `/root` | ✅ Total | ✅ Total |
| `admin-web` | Carles | Administrador Web | `/var/www/html` | ❌ | ❌ |
| `db-backup` | Michael | Seguridad / Backups | `/var/backups/mysql` | ❌ | ✅ Lectura |
| `sys-monitor` | Javi | Auditor / Monitorización | `/var/log` (lectura) | ❌ | ❌ |

---

!!! note "Verificación global de usuarios"
    Para ver todos los usuarios creados en el sistema:
    ```bash
    cat /etc/passwd | grep -v nologin | grep -v false
    ```
    Para ver los grupos de cada usuario:
    ```bash
    for user in admin-web db-backup sys-monitor; do echo "$user: $(groups $user)"; done
    ```
