# 🔒 Seguretat i Hardening del Servidor

En aquesta secció es documenten totes les mesures de seguretat aplicades al servidor per reduir la superfície d'atac i protegir els serveis desplegats. L'**hardening** (enduriment) és el procés de configurar el sistema per eliminar vulnerabilitats innecessàries.

---

## 🛡️ Pas 1: Configuració del Firewall amb UFW

**UFW** (*Uncomplicated Firewall*) és la ferramenta de gestió de firewall per a Ubuntu/Debian. Permet definir regles senzilles per controlar el tràfic de xarxa entrant i eixint del servidor.

### Comprovació de l'estat inicial

```bash
sudo ufw status
```

Si el firewall no està actiu, la resposta serà `Status: inactive`.

### Política per defecte: denegar tot

Abans d'afegir regles, establim una política restrictiva que **bloqueja tot el tràfic entrant** i permet tot el sortint:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

!!! warning "Important: ordre de configuració"
    **Sempre afegeix les regles necessàries ABANS d'activar el firewall.** Si actives UFW sense permetre SSH, perdràs l'accés remot al servidor.

### Permetre SSH (connexions remotes)

Permetem l'accés SSH al servidor. Com hem canviat el port per defecte a **2222** (veure secció [Canvi de Port SSH](#pas-2-hardening-del-servei-ssh)), hem d'especificar-lo:

```bash
sudo ufw allow ssh
```

O directament per port, en cas d'haver modificat el port al 2222:

```bash
sudo ufw allow 2222/tcp
```

### Permetre HTTP (tràfic web)

Permetem el tràfic web entrant al port 80 (HTTP):

```bash
sudo ufw allow 80/tcp
```

Si en el futur s'afig un certificat SSL, s'haurà de permetre també el port 443 (HTTPS):

```bash
sudo ufw allow 443/tcp
```

### Activació del Firewall

Un cop configurades les regles, activem UFW:

```bash
sudo ufw enable
```

El sistema demanarà confirmació. Escrivim `y` i premem `Enter`.

### Verificació de les regles actives

```bash
sudo ufw status verbose
```

La sortida esperada és similar a:

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

!!! success "Firewall actiu"
    El servidor ara bloqueja tot el tràfic entrant excepte SSH (port 2222) i HTTP (port 80). Qualsevol altre port no inclòs en les regles serà rebutjat automàticament.

---

## 🔑 Pas 2: Hardening del Servei SSH

SSH (*Secure Shell*) és el protocol principal d'accés remot al servidor. La seua configuració per defecte presenta diverses vulnerabilitats que s'han de corregir.

### Edició del fitxer de configuració SSH

El fitxer de configuració principal d'SSH és `/etc/ssh/sshd_config`. L'editem com a root:

```bash
sudo nano /etc/ssh/sshd_config
```

### 🔄 Canvi del Port per Defecte

El port per defecte d'SSH és el **22**, que és escanejat constantment per bots i atacants automatitzats. Canviem-lo al **2222** per reduir la seua exposició.

Localitzem la línia:

```
#Port 22
```

I la substituïm per:

```
Port 2222
```

!!! info "Per què canviar el port?"
    Canviar el port SSH **no és una mesura de seguretat definitiva** per si sola (seguretat per obscuritat), però redueix significativament el soroll dels atacs automatitzats de força bruta, que solen apuntar sempre al port 22. S'ha de combinar amb altres mesures com la desactivació de `root` i les claus SSH.

### 🚫 Desactivació del Login Directe com a Root

Localitzem la línia:

```
#PermitRootLogin prohibit-password
```

I la substituïm per:

```
PermitRootLogin no
```

Açò impedeix que ningú puga connectar-se directament com a `root` per SSH, obligant a usar un compte d'usuari i després elevar privilegis amb `sudo`.

### ⏱️ Limitació d'Intents d'Autenticació

Limitem el nombre d'intents de contrasenya per connexió:

```
MaxAuthTries 3
```

### 🕐 Temps Màxim de Login

Establim un temps límit per completar l'autenticació:

```
LoginGraceTime 30
```

### 📝 Desactivació de l'Autenticació per Contrasenya (Recomanat)

Si s'utilitzen **claus SSH**, es pot desactivar l'autenticació per contrasenya completament per evitar atacs de força bruta:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

!!! danger "Atenció abans de desactivar contrasenyes"
    **Assegura't de tindre les teues claus SSH correctament configurades** i provades abans de desactivar l'autenticació per contrasenya. Si perds l'accés, necessitaràs accés físic o per consola al servidor.

### Desament i reinici del servei SSH

Guardem els canvis amb `Ctrl + O`, `Enter` i sortim amb `Ctrl + X`.

Reiniciem el servei SSH per aplicar els canvis:

```bash
sudo systemctl restart sshd
```

Verifiquem que SSH escolta ara al port 2222:

```bash
sudo ss -tlnp | grep sshd
```

La sortida hauria de mostrar `*:2222` en comptes de `*:22`.

!!! warning "Connexions SSH futures"
    A partir d'ara, per connectar-se al servidor per SSH, s'haurà d'especificar el port:
    ```bash
    ssh -p 2222 usuari@IP_DEL_SERVIDOR
    ```

---

## 🔍 Pas 3: Mesures Addicionals de Seguretat

### Instal·lació de Fail2Ban

**Fail2Ban** monitoritza els fitxers de log i bloqueja automàticament les IPs que superen un nombre d'intents fallits d'autenticació:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Verifiquem el seu estat:

```bash
sudo fail2ban-client status
```

### Desactivació de serveis innecessaris

Llistem els serveis actius per identificar possibles candidats a desactivar:

```bash
sudo systemctl list-units --type=service --state=running
```

### Actualitzacions automàtiques de seguretat

Instal·lem el paquet d'actualitzacions de seguretat desateses:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 📋 Resum de Mesures de Seguretat Aplicades

| Mesura | Estat | Detall |
|--------|:-----:|--------|
| Firewall UFW actiu | ✅ | Política `deny incoming` per defecte |
| Port SSH canviat | ✅ | Port 22 → Port **2222** |
| Login root per SSH desactivat | ✅ | `PermitRootLogin no` |
| Màxim d'intents SSH limitat | ✅ | `MaxAuthTries 3` |
| HTTP permès | ✅ | Port 80/tcp obert |
| Fail2Ban instal·lat | ✅ | Bloqueig automàtic d'IPs sospitoses |
| Actualitzacions automàtiques | ✅ | `unattended-upgrades` actiu |
| Accés remot root desactivat | ✅ | `PermitRootLogin no` en sshd_config |

---

!!! note "Revisió periòdica"
    La seguretat no és un estat, és un **procés continu**. Cal revisar regularment:

    - Els logs de Fail2Ban: `sudo fail2ban-client status sshd`
    - Les regles del firewall: `sudo ufw status verbose`
    - Els logs d'autenticació: `sudo tail -f /var/log/auth.log`
    - Les actualitzacions pendents: `sudo apt list --upgradable`
