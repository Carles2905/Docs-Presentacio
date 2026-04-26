# 👤 Gestió de Rols i Usuaris

En aquesta secció es documenta la creació i configuració de tots els usuaris del sistema, aplicant el **Principi de Mínim Privilegi** (*Principle of Least Privilege*): cada usuari disposa únicament dels permisos imprescindibles per realitzar les seues funcions i res més.

---

## 🔐 Principi de Mínim Privilegi

El principi de mínim privilegi estableix que **cap usuari, procés o servei ha de tenir més permisos dels estrictament necessaris** per a la seua funció. Açò redueix la superfície d'atac del sistema: si un compte és compromès, el dany possible queda limitat als recursos als quals aquell usuari tenia accés.

---

## 👑 Usuari: `root` — Adrián (Scrum Master)

### Descripció

L'usuari `root` és el superusuari del sistema Linux amb **accés il·limitat** a tots els fitxers, serveis i configuracions. En el nostre projecte, aquest rol correspon a **Adrián**, que actua com a Scrum Master i responsable tècnic global.

### Permisos

- ✅ Accés total a tots els directoris del sistema
- ✅ Capacitat per instal·lar i desinstal·lar paquets
- ✅ Gestió de serveis amb `systemctl`
- ✅ Creació i eliminació d'usuaris
- ✅ Modificació de qualsevol fitxer de configuració

### Consideracions de Seguretat

!!! danger "Ús responsable del root"
    L'accés com a `root` ha d'usar-se **únicament per a tasques que ho requerisquen explícitament**. Per a tasques quotidianes s'hauria d'usar `sudo` des d'un compte personal amb privilegis limitats.

L'accés SSH directe com a `root` està **desactivat** en la configuració de seguretat (veure secció [Seguretat i Hardening](seguretat.md)).

---

## 🌐 Usuari: `admin-web` — Carles (Administrador Web)

### Descripció

L'usuari `admin-web` és el responsable del servidor web Apache. **Carles** gestiona els fitxers de l'aplicació web, els *virtual hosts* i el contingut públic del servidor, sense tenir accés a la base de dades ni als logs del sistema.

### Creació de l'usuari

```bash
sudo adduser admin-web
```

Durant la creació, s'assignarà una contrasenya i s'ompliran les dades opcionals (nom complet, etc.).

### Assignació de propietat del directori web

L'usuari `admin-web` ha de ser el **propietari** del directori `/var/www/html` per poder gestionar els fitxers de la web sense necessitar permisos de `root`:

```bash
sudo chown -R admin-web:admin-web /var/www/html
```

Ajustem els permisos del directori perquè siguen correctes:

```bash
sudo chmod -R 755 /var/www/html
```

Verifiquem que la propietat s'ha assignat correctament:

```bash
ls -la /var/www/html
```

### Permisos

| Recurs | Permís |
|--------|--------|
| `/var/www/html` | ✅ Lectura i escriptura (propietari) |
| `/etc/apache2` | ❌ Sense accés |
| `/var/log` | ❌ Sense accés |
| MariaDB | ❌ Sense accés |
| `sudo` | ❌ No té privilegis sudo |

### Canvi d'identitat per operar

Per treballar com a `admin-web`, Adrián (root) pot fer:

```bash
sudo su - admin-web
```

---

## 🗄️ Usuari: `db-backup` — Michael (Seguretat i Backups)

### Descripció

L'usuari `db-backup` té com a única funció la realització de **còpies de seguretat de les bases de dades** de MariaDB. **Michael** és el responsable de configurar i executar els scripts de `mysqldump`. No té accés al sistema de fitxers web ni als logs del sistema.

### Creació de l'usuari del sistema

```bash
sudo adduser --system --no-create-home --shell /bin/bash db-backup
```

L'opció `--system` crea un usuari del sistema sense directori personal, adequat per a tasques automatitzades.

### Creació de l'usuari a MariaDB

Creem un usuari a MariaDB amb permisos de lectura per a la realització de còpies:

```bash
sudo mysql -u root -p
```

Un cop dins del prompt de MariaDB:

```sql
CREATE USER 'db-backup'@'localhost' IDENTIFIED BY 'BackupPass_2024!';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'db-backup'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Script de còpia de seguretat

Creem el directori on es guardaran les còpies:

```bash
sudo mkdir -p /var/backups/mysql
sudo chown db-backup:db-backup /var/backups/mysql
```

Creem el script de còpia automàtica:

```bash
sudo nano /usr/local/bin/backup-db.sh
```

Contingut del script:

```bash
#!/bin/bash
DATA=$(date +%Y-%m-%d_%H-%M-%S)
DEST="/var/backups/mysql"
mysqldump -u db-backup -pBackupPass_2024! --all-databases > "$DEST/backup_$DATA.sql"
echo "Còpia realitzada: backup_$DATA.sql"
```

Donem permisos d'execució al script:

```bash
sudo chmod +x /usr/local/bin/backup-db.sh
sudo chown db-backup:db-backup /usr/local/bin/backup-db.sh
```

### Permisos

| Recurs | Permís |
|--------|--------|
| `/var/backups/mysql` | ✅ Lectura i escriptura (propietari) |
| MariaDB (lectura) | ✅ `SELECT`, `LOCK TABLES` sobre totes les bases de dades |
| `/var/www/html` | ❌ Sense accés |
| `/var/log` | ❌ Sense accés |
| `sudo` | ❌ No té privilegis sudo |

---

## 📊 Usuari: `sys-monitor` — Javi (Auditor del Sistema)

### Descripció

L'usuari `sys-monitor` és l'encarregat de l'**auditoria i monitoratge del servidor**. **Javi** revisa els fitxers de log del sistema per detectar errors, accessos no autoritzats o comportaments anòmals. Té permisos de **només lectura** i no pot modificar cap configuració.

### Creació de l'usuari

```bash
sudo adduser sys-monitor
```

### Configuració de permisos de lectura als logs

Afegim `sys-monitor` al grup `adm`, que en Ubuntu/Debian té accés de lectura als logs del sistema:

```bash
sudo usermod -aG adm sys-monitor
```

Verifiquem que l'usuari pertany al grup `adm`:

```bash
groups sys-monitor
```

### Verificació d'accés als logs

L'usuari `sys-monitor` podrà llegir fitxers com:

```bash
# Logs d'autenticació (SSH, sudo, etc.)
tail -n 100 /var/log/auth.log

# Logs generals del sistema
tail -n 100 /var/log/syslog

# Logs d'Apache
tail -n 100 /var/log/apache2/access.log
tail -n 100 /var/log/apache2/error.log
```

### Restriccions addicionals

Restricció de l'accés al directori `/var/www/html`:

```bash
sudo chmod o-rx /var/www/html
```

### Permisos

| Recurs | Permís |
|--------|--------|
| `/var/log` | ✅ Lectura (membre del grup `adm`) |
| `/var/log/apache2` | ✅ Lectura |
| `/var/www/html` | ❌ Sense accés |
| MariaDB | ❌ Sense accés |
| `sudo` | ❌ No té privilegis sudo |

---

## 📋 Resum de Tots els Usuaris

| Usuari | Membre de l'equip | Rol | Directori principal | Accés sudo | Accés MariaDB |
|--------|-------------------|-----|--------------------|-----------:|:-------------:|
| `root` | Adrián | Scrum Master / Admin total | `/root` | ✅ Total | ✅ Total |
| `admin-web` | Carles | Administrador Web | `/var/www/html` | ❌ | ❌ |
| `db-backup` | Michael | Seguretat / Backups | `/var/backups/mysql` | ❌ | ✅ Lectura |
| `sys-monitor` | Javi | Auditor / Monitoratge | `/var/log` (lectura) | ❌ | ❌ |

---

!!! note "Verificació global d'usuaris"
    Per veure tots els usuaris creats al sistema:
    ```bash
    cat /etc/passwd | grep -v nologin | grep -v false
    ```
    Per veure els grups de cada usuari:
    ```bash
    for user in admin-web db-backup sys-monitor; do echo "$user: $(groups $user)"; done
    ```
