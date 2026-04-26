# ⚙️ Configuración de Servidores — Pila LAMP

En esta sección se documenta el proceso completo de instalación y configuración de la pila **LAMP** sobre un servidor **Ubuntu 22.04 LTS**. Todos los comandos deben ejecutarse como usuario `root` o con `sudo`.

---

## 🔄 Paso 1: Actualización del Sistema

Antes de cualquier instalación, es imprescindible actualizar el índice de paquetes de los repositorios y actualizar los paquetes instalados en el sistema para garantizar que todos los componentes estén en sus versiones más recientes y seguras.

```bash
sudo apt update && sudo apt upgrade -y
```

!!! tip "Buena práctica"
    Realiza siempre `apt update` y `apt upgrade` antes de instalar cualquier paquete nuevo. Esto evita conflictos de dependencias y asegura que el sistema tenga los últimos parches de seguridad.

---

## 🌐 Paso 2: Instalación de Apache2

**Apache2** es el servidor web que gestionará las peticiones HTTP y servirá el contenido de la aplicación a los clientes.

```bash
sudo apt install apache2 -y
```

Una vez instalado, verificamos que el servicio esté activo y en marcha:

```bash
sudo systemctl status apache2
```

Habilitamos Apache para que se inicie automáticamente con el sistema:

```bash
sudo systemctl enable apache2
```

Comprobamos que Apache responde correctamente en el puerto 80:

```bash
curl -I http://localhost
```

El resultado esperado es una respuesta con `HTTP/1.1 200 OK` y las cabeceras de Apache.

!!! success "Verificación"
    Si accedes desde un navegador a la IP del servidor, deberías ver la página por defecto de Apache2: **"Apache2 Ubuntu Default Page"**.

---

## 🗄️ Paso 3: Instalación de MariaDB

**MariaDB** es el sistema gestor de bases de datos relacional (SGBD) que almacenará la información de la aplicación web.

```bash
sudo apt install mariadb-server -y
```

Iniciamos el servicio y lo habilitamos para el inicio automático:

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

Verificamos el estado del servicio:

```bash
sudo systemctl status mariadb
```

### 🔐 Securización de MariaDB

Ejecutamos el asistente de seguridad integrado para eliminar los usuarios anónimos, desactivar el acceso remoto de `root` y borrar la base de datos de prueba:

```bash
sudo mysql_secure_installation
```

Durante el asistente, responderemos:

- `Enter current password for root`: dejar en blanco y pulsar `Enter`
- `Switch to unix_socket authentication`: `n`
- `Change the root password?`: `y` → introducir una contraseña segura
- `Remove anonymous users?`: `y`
- `Disallow root login remotely?`: `y`
- `Remove test database and access to it?`: `y`
- `Reload privilege tables now?`: `y`

Verificamos el acceso a MariaDB:

```bash
sudo mysql -u root -p
```

!!! warning "Seguridad de la contraseña"
    Utiliza una contraseña robusta para `root` de MariaDB: mínimo 12 caracteres, combinando mayúsculas, minúsculas, números y símbolos.

---

## 🐘 Paso 4: Instalación de PHP

**PHP** es el lenguaje de programación del lado del servidor que permitirá la ejecución dinámica del código de la aplicación web. Instalamos los módulos principales y las extensiones necesarias para la integración con Apache y MariaDB.

```bash
sudo apt install php libapache2-mod-php php-mysql -y
```

Verificamos la versión de PHP instalada:

```bash
php --version
```

### ✅ Prueba de Funcionamiento de PHP

Creamos un fichero de prueba para verificar que Apache procesa correctamente el PHP:

```bash
sudo nano /var/www/html/info.php
```

Añadimos el siguiente contenido al fichero:

```php
<?php
phpinfo();
?>
```

Guarda el fichero con `Ctrl + O`, `Enter` y sal con `Ctrl + X`.

Reiniciamos Apache para aplicar los cambios del módulo PHP:

```bash
sudo systemctl restart apache2
```

Accede desde un navegador a `http://IP_DEL_SERVIDOR/info.php`. Si ves la página de información de PHP, la instalación ha sido correcta.

!!! danger "Elimina el fichero de prueba"
    Una vez verificada la instalación, **elimina el fichero `info.php`** inmediatamente, ya que expone información sensible del servidor:
    ```bash
    sudo rm /var/www/html/info.php
    ```

---

## 📊 Resumen de la Pila LAMP

| Componente | Paquete instalado | Puerto por defecto | Servicio systemd |
|-----------|-------------------|-------------------|-----------------|
| **L**inux | Ubuntu 22.04 LTS | — | — |
| **A**pache | `apache2` | 80 (HTTP) | `apache2` |
| **M**ariaDB | `mariadb-server` | 3306 | `mariadb` |
| **P**HP | `php libapache2-mod-php php-mysql` | — | (módulo de Apache) |

---

!!! note "Orden de instalación"
    El orden recomendado es siempre: **Apache → MariaDB → PHP**. Instalar PHP antes que Apache puede causar que el módulo `libapache2-mod-php` no se configure correctamente.
