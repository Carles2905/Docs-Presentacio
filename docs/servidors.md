# ⚙️ Configuració de Servidors — Pila LAMP

En aquesta secció es documenta el procés complet d'instal·lació i configuració de la pila **LAMP** sobre un servidor **Ubuntu 22.04 LTS**. Tots els comandos s'han d'executar com a usuari `root` o amb `sudo`.

---

## 🔄 Pas 1: Actualització del Sistema

Abans de qualsevol instal·lació, és imprescindible actualitzar l'índex de paquets dels repositoris i actualitzar els paquets instal·lats al sistema per garantir que tots els components estiguen en les seues versions més recents i segures.

```bash
sudo apt update && sudo apt upgrade -y
```

!!! tip "Bona pràctica"
    Sempre realitza `apt update` i `apt upgrade` abans d'instal·lar qualsevol paquet nou. Açò evita conflictes de dependències i assegura que el sistema tinga els últims pedaços de seguretat.

---

## 🌐 Pas 2: Instal·lació d'Apache2

**Apache2** és el servidor web que gestionarà les peticions HTTP i servirà el contingut de l'aplicació als clients.

```bash
sudo apt install apache2 -y
```

Un cop instal·lat, verifiquem que el servei estiga actiu i en marxa:

```bash
sudo systemctl status apache2
```

Habilitem Apache perquè s'inicie automàticament amb el sistema:

```bash
sudo systemctl enable apache2
```

Comprovem que Apache respon correctament al port 80:

```bash
curl -I http://localhost
```

El resultat esperat és una resposta amb `HTTP/1.1 200 OK` i les capçaleres d'Apache.

!!! success "Verificació"
    Si accedeixes des d'un navegador a la IP del servidor, hauries de veure la pàgina per defecte d'Apache2: **"Apache2 Ubuntu Default Page"**.

---

## 🗄️ Pas 3: Instal·lació de MariaDB

**MariaDB** és el sistema gestor de bases de dades relacional (SGBD) que emmagatzemarà la informació de l'aplicació web.

```bash
sudo apt install mariadb-server -y
```

Iniciem el servei i l'habilitem per a l'inici automàtic:

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

Verifiquem l'estat del servei:

```bash
sudo systemctl status mariadb
```

### 🔐 Securització de MariaDB

Executem l'assistent de seguretat integrat per eliminar els usuaris anònims, desactivar l'accés remot de `root` i esborrar la base de dades de prova:

```bash
sudo mysql_secure_installation
```

Durant l'assistent, respondrem:

- `Enter current password for root`: deixar en blanc i prémer `Enter`
- `Switch to unix_socket authentication`: `n`
- `Change the root password?`: `y` → introduir una contrasenya segura
- `Remove anonymous users?`: `y`
- `Disallow root login remotely?`: `y`
- `Remove test database and access to it?`: `y`
- `Reload privilege tables now?`: `y`

Verifiquem l'accés a MariaDB:

```bash
sudo mysql -u root -p
```

!!! warning "Seguretat de la contrasenya"
    Utilitza una contrasenya robusta per a `root` de MariaDB: com a mínim 12 caràcters, combinant majúscules, minúscules, números i símbols.

---

## 🐘 Pas 4: Instal·lació de PHP

**PHP** és el llenguatge de programació del costat del servidor que permetrà l'execució dinàmica del codi de l'aplicació web. Instal·lem els mòduls principals i les extensions necessàries per a la integració amb Apache i MariaDB.

```bash
sudo apt install php libapache2-mod-php php-mysql -y
```

Verifiquem la versió de PHP instal·lada:

```bash
php --version
```

### ✅ Prova de Funcionament de PHP

Creem un fitxer de prova per verificar que Apache processa correctament el PHP:

```bash
sudo nano /var/www/html/info.php
```

Afegim el contingut següent al fitxer:

```php
<?php
phpinfo();
?>
```

Desa el fitxer amb `Ctrl + O`, `Enter` i surt amb `Ctrl + X`.

Reiniciem Apache per aplicar els canvis del mòdul PHP:

```bash
sudo systemctl restart apache2
```

Accedeix des d'un navegador a `http://IP_DEL_SERVIDOR/info.php`. Si veus la pàgina d'informació de PHP, la instal·lació ha estat correcta.

!!! danger "Elimina el fitxer de prova"
    Un cop verificada la instal·lació, **elimina el fitxer `info.php`** immediatament, ja que exposa informació sensible del servidor:
    ```bash
    sudo rm /var/www/html/info.php
    ```

---

## 📊 Resum de la Pila LAMP

| Component | Paquet instal·lat | Port per defecte | Servei systemd |
|-----------|-------------------|-----------------|----------------|
| **L**inux | Ubuntu 22.04 LTS | — | — |
| **A**pache | `apache2` | 80 (HTTP) | `apache2` |
| **M**ariaDB | `mariadb-server` | 3306 | `mariadb` |
| **P**HP | `php libapache2-mod-php php-mysql` | — | (mòdul d'Apache) |

---

!!! note "Ordre d'instal·lació"
    L'ordre recomanat és sempre: **Apache → MariaDB → PHP**. Instal·lar PHP abans d'Apache pot causar que el mòdul `libapache2-mod-php` no es configure correctament.
