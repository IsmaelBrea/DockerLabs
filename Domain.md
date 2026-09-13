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


Vamos a acceder al recurso /html:






