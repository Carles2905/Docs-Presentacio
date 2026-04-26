# 🔒 Seguridad y Hardening del Servidor

En esta sección se documentan todas las medidas de seguridad aplicadas al servidor para reducir la superficie de ataque y proteger los servicios desplegados. El **hardening** (endurecimiento) es el proceso de configurar el sistema para eliminar vulnerabilidades innecesarias.

---

## 🛡️ Paso 1: Configuración del Firewall con UFW

**UFW** (*Uncomplicated Firewall*) es la herramienta de gestión de firewall para Ubuntu/Debian. Permite definir reglas sencillas para controlar el tráfico de red entrante y saliente del servidor.

### Comprobación del estado inicial

```bash
sudo ufw status
```

Si el firewall no está activo, la respuesta será `Status: inactive`.

### Política por defecto: denegar todo

Antes de añadir reglas, establecemos una política restrictiva que **bloquea todo el tráfico entrante** y permite todo el saliente:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

!!! warning "Importante: orden de configuración"
    **Añade siempre las reglas necesarias ANTES de activar el firewall.** Si activas UFW sin permitir SSH, perderás el acceso remoto al servidor.

### Permitir SSH (conexiones remotas)

Permitimos el acceso SSH al servidor. Como hemos cambiado el puerto por defecto a **2222** (ver sección [Cambio de Puerto SSH](#paso-2-hardening-del-servicio-ssh)), debemos especificarlo:

```bash
sudo ufw allow ssh
```

O directamente por puerto, en caso de haber modificado el puerto a 2222:

```bash
sudo ufw allow 2222/tcp
```

### Permitir HTTP (tráfico web)

Permitimos el tráfico web entrante en el puerto 80 (HTTP):

```bash
sudo ufw allow 80/tcp
```

Si en el futuro se añade un certificado SSL, habrá que permitir también el puerto 443 (HTTPS):

```bash
sudo ufw allow 443/tcp
```

### Activación del Firewall

Una vez configuradas las reglas, activamos UFW:

```bash
sudo ufw enable
```

El sistema pedirá confirmación. Escribimos `y` y pulsamos `Enter`.

### Verificación de las reglas activas

```bash
sudo ufw status verbose
```

La salida esperada es similar a:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

!!! success "Firewall activo"
    El servidor ahora bloquea todo el tráfico entrante excepto SSH (puerto 2222) y HTTP (puerto 80). Cualquier otro puerto no incluido en las reglas será rechazado automáticamente.

---

## 🔑 Paso 2: Hardening del Servicio SSH

SSH (*Secure Shell*) es el protocolo principal de acceso remoto al servidor. Su configuración por defecto presenta varias vulnerabilidades que deben corregirse.

### Edición del fichero de configuración SSH

El fichero de configuración principal de SSH es `/etc/ssh/sshd_config`. Lo editamos como root:

```bash
sudo nano /etc/ssh/sshd_config
```

### 🔄 Cambio del Puerto por Defecto

El puerto por defecto de SSH es el **22**, que es escaneado constantemente por bots y atacantes automatizados. Lo cambiamos al **2222** para reducir su exposición.

Localizamos la línea:

```
#Port 22
```

Y la sustituimos por:

```
Port 2222
```

!!! info "¿Por qué cambiar el puerto?"
    Cambiar el puerto SSH **no es una medida de seguridad definitiva** por sí sola (seguridad por oscuridad), pero reduce significativamente el ruido de los ataques automatizados de fuerza bruta, que suelen apuntar siempre al puerto 22. Debe combinarse con otras medidas como la desactivación de `root` y las claves SSH.

### 🚫 Desactivación del Login Directo como Root

Localizamos la línea:

```
#PermitRootLogin prohibit-password
```

Y la sustituimos por:

```
PermitRootLogin no
```

Esto impide que nadie pueda conectarse directamente como `root` por SSH, obligando a usar una cuenta de usuario y después elevar privilegios con `sudo`.

### ⏱️ Limitación de Intentos de Autenticación

Limitamos el número de intentos de contraseña por conexión:

```
MaxAuthTries 3
```

### 🕐 Tiempo Máximo de Login

Establecemos un tiempo límite para completar la autenticación:

```
LoginGraceTime 30
```

### 📝 Desactivación de la Autenticación por Contraseña (Recomendado)

Si se utilizan **claves SSH**, se puede desactivar la autenticación por contraseña completamente para evitar ataques de fuerza bruta:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

!!! danger "Atención antes de desactivar contraseñas"
    **Asegúrate de tener tus claves SSH correctamente configuradas** y probadas antes de desactivar la autenticación por contraseña. Si pierdes el acceso, necesitarás acceso físico o por consola al servidor.

### Guardado y reinicio del servicio SSH

Guardamos los cambios con `Ctrl + O`, `Enter` y salimos con `Ctrl + X`.

Reiniciamos el servicio SSH para aplicar los cambios:

```bash
sudo systemctl restart sshd
```

Verificamos que SSH escucha ahora en el puerto 2222:

```bash
sudo ss -tlnp | grep sshd
```

La salida debería mostrar `*:2222` en lugar de `*:22`.

!!! warning "Conexiones SSH futuras"
    A partir de ahora, para conectarse al servidor por SSH, habrá que especificar el puerto:
    ```bash
    ssh -p 2222 usuario@IP_DEL_SERVIDOR
    ```

---

## 🔍 Paso 3: Medidas Adicionales de Seguridad

### Instalación de Fail2Ban

**Fail2Ban** monitoriza los ficheros de log y bloquea automáticamente las IPs que superan un número de intentos fallidos de autenticación:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Verificamos su estado:

```bash
sudo fail2ban-client status
```

### Desactivación de servicios innecesarios

Listamos los servicios activos para identificar posibles candidatos a desactivar:

```bash
sudo systemctl list-units --type=service --state=running
```

### Actualizaciones automáticas de seguridad

Instalamos el paquete de actualizaciones de seguridad desatendidas:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 📋 Resumen de Medidas de Seguridad Aplicadas

| Medida | Estado | Detalle |
|--------|:-----:|---------|
| Firewall UFW activo | ✅ | Política `deny incoming` por defecto |
| Puerto SSH cambiado | ✅ | Puerto 22 → Puerto **2222** |
| Login root por SSH desactivado | ✅ | `PermitRootLogin no` |
| Máximo de intentos SSH limitado | ✅ | `MaxAuthTries 3` |
| HTTP permitido | ✅ | Puerto 80/tcp abierto |
| Fail2Ban instalado | ✅ | Bloqueo automático de IPs sospechosas |
| Actualizaciones automáticas | ✅ | `unattended-upgrades` activo |
| Acceso remoto root desactivado | ✅ | `PermitRootLogin no` en sshd_config |

---

!!! note "Revisión periódica"
    La seguridad no es un estado, es un **proceso continuo**. Hay que revisar regularmente:

    - Los logs de Fail2Ban: `sudo fail2ban-client status sshd`
    - Las reglas del firewall: `sudo ufw status verbose`
    - Los logs de autenticación: `sudo tail -f /var/log/auth.log`
    - Las actualizaciones pendientes: `sudo apt list --upgradable`
