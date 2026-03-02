# 🏴 OverTheWire: Bandit — Writeup (Niveles 0–12)

> Documentación personal del progreso en el wargame [Bandit](https://overthewire.org/wargames/bandit/) de OverTheWire.  
> Objetivo: aprender conceptos básicos de Linux, SSH y seguridad.

---

## Conexión general

```bash
ssh bandit<NIVEL>@bandit.labs.overthewire.org -p 2220
```

---

## Bandit 1

Solo hicimos un `cat` xd

```bash
cat readme
```

**Contraseña:** `ZjLjTmM6FvvyRnrb2rfNWOZOTa6ip5If`

---

## Bandit 2

Usamos `cat ./-` para poder ver los archivos que tienen un guion como nombre.

```bash
cat ./-
```

**Contraseña:** `263JGJPfgU6LtdEvgfWU1XP5yac29mFx`

---

## Bandit 3

Al poner `./` delante, le indicas que el archivo está en el directorio actual. Como ya no empieza por guion, `cat` no se confunde.

```bash
cat "./--spaces in this filename--"
```

**Contraseña:** `MNk8KNH3Usiio41PRUEoDFPqfxLPlSmx`

---

## Bandit 4

Solo nos movimos de carpeta y con un `cat` al archivo oculto:

```bash
cat ..Hiding-From-You
```

**Contraseña:** `2WmrDFRmJIq3IPxneAaMGhap0pFhF3NJ`

---

## Bandit 5

Aquí sí nos moveremos hasta la carpeta `inhere` y usaremos `find` combinado con `xargs file` para identificar cuál archivo es texto legible:

```bash
find . -type f | xargs file
```

**Contraseña:** `4oQYVPkxZOOEOO5pTW81FB8j8lxXGUQw`

---

## Bandit 6

Esta sí estuvo más interesante, sacamos a pasear a nuestro `find` buscando por tamaño:

```bash
find . -type f -size 1033c
```

**Contraseña:** `HWasnPhtq9AVKe0dmk45nxy20cvUa6EG`

---

## Bandit 7

En este caso también sacamos a pasear nuestro `find` xd, pero ahora buscando por usuario, grupo y tamaño en todo el sistema:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c
```

**Contraseña:** `morbNTDkSW6jIlUc0ymOdMaLnOlFVAaj`

---

## Bandit 8

Solo usamos un `grep` para encontrar la línea junto a la palabra `millionth`:

```bash
cat data.txt | grep millionth
```

**Contraseña:** `dfwvzFQi4mU0wfNbFOe9RoWskMLg7eEc`

---

## Bandit 9

Usaremos nuestro poderoso `sort`, que es para ordenar, y luego `uniq -c` que cuenta cuántas veces se repite cada línea:

```bash
sort data.txt | uniq -c
```

> La línea que aparece solo una vez es la que contiene la contraseña.

**Contraseña:** `4CKMh1JI91bUIZZPXDqGanal4xvAg0JM`

---

## Bandit 10

En este caso buscamos palabras legibles en el binario con `strings` y filtramos por `=`:

```bash
strings data.txt | grep "="
```

**Contraseña:** `FGUW5ilLVJrxX9kMYMmlN4MgbpfMiqey`

---

## Bandit 11

En este caso decodificamos la base 64:

```bash
base64 -d data.txt
```

**Contraseña:** `dtR173fZKb0RRsDFSGsg2RWnpNVj3qRr`

---

## Bandit 12

Nos piden rotar 13 posiciones (ROT13). Nosotros nos sacamos las posiciones pero del Kamasutra así que metemos full Google.

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

> ROT13 simplemente mueve cada letra 13 posiciones en el alfabeto. Como el alfabeto tiene 26 letras, aplicarlo dos veces te devuelve al original.

**Contraseña:** `7x16WNeHIi5YkIhWsfFIqoognUTyj9Q4`

---

## Resumen de comandos aprendidos

| Comando | Qué hace |
|---------|----------|
| `cat ./-` | Leer archivos cuyo nombre empieza con `-` |
| `cat "./nombre"` | Leer archivos con espacios o nombres raros |
| `find . -type f \| xargs file` | Identificar tipo de cada archivo |
| `find / -type f -user X -group Y -size Zc` | Buscar archivos por dueño, grupo y tamaño |
| `grep palabra archivo` | Buscar una palabra dentro de un archivo |
| `sort \| uniq -c` | Ordenar y contar repeticiones de líneas |
| `strings archivo \| grep "="` | Extraer texto legible de binarios |
| `base64 -d archivo` | Decodificar Base64 |
| `tr 'A-Za-z' 'N-ZA-Mn-za-m'` | Aplicar ROT13 |

---

*Wargame: [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) — Progreso personal*