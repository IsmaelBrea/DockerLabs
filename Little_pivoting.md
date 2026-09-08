# Máquina Trust

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







