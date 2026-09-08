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

 ### Gobuster
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

Por tanto, usaremos la máquina trust como pivote para acceder desde la Kali a la 20.20.20.2.

Usaremos chisel.

Como en Trust no tenemos chisel, tenemos que pasarle el binario a la máquina desde Kali.


