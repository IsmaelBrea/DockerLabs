# Máquina Walking_Dead

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
sudo bash auto_deploy.sh walkingdead.tar
```

--- 

Como siempre empezamos escaneando la máquina para ver servicios abiertos:
```bash
nmap -sCV -p- --open --min-rate 5000 172.17.0.2 -vvv
```

Tenemos 2 servicios abiertos: 22 y 80:
<img width="1594" height="529" alt="imagen" src="https://github.com/user-attachments/assets/324a06ec-33bf-434c-9c7a-df614c2d7fb2" />

Vamos a empezar por ver y enumerar la web:

Nos encontramos esto en la web:
<img width="1579" height="532" alt="imagen" src="https://github.com/user-attachments/assets/673773c4-3b77-4075-9f2c-d399c17c8094" />

Nada relevante de primeras. Vamos a fuzzearla:
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt 
```

Encontramos solo una ruta:
<img width="1119" height="595" alt="imagen" src="https://github.com/user-attachments/assets/43d05479-d1e1-4e6f-b658-3da5f3e3ac4d" />

Pero en ella no encontramos nada.

Fuzzing más específico:
```bash
gobuster dir -u http://172.17.0.2 \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt,html,bak,old
```

Encontramos un `index.html`. Ahora si vemos una web:

<img width="1540" height="568" alt="imagen" src="https://github.com/user-attachments/assets/5f61524e-c530-492f-8c50-5ea2123c6cb1" />

Vamos a ver el código fuente (Ctrl+U):
<img width="1582" height="121" alt="imagen" src="https://github.com/user-attachments/assets/f8e241b0-b073-4ad1-8fe1-f0a11466ef6b" />


En el HTML hay un enlace oculto:
```html
<a href="hidden/.shell.php">Access Panel</a>
```
Está oculto con display: none, pero apunta a: `/hidden/.shell.php`

Por tanto, la pista importante es .shell.php. Vamos a comprobar que funciona:
```bash
curl -v http://172.17.0.2/hidden/.shell.php
```

<img width="1248" height="460" alt="imagen" src="https://github.com/user-attachments/assets/d50e1148-3e63-4b37-8359-1aa95dd3bc82" />

Perfecto. Esto nos confirma algo importante:

- /.shell.php existe → HTTP 200 OK
- Apache lo está procesando como PHP.
- Content-Length: 0 → no devuelve ningún contenido directamente.

Por tanto, probablemente espera algún parámetro o realiza alguna acción sin mostrar salida. Lo siguiente que probaría es revisar cómo responde ante parámetros comunes. Podemos hacerlo con curl o en el navegador:

<img width="1217" height="132" alt="imagen" src="https://github.com/user-attachments/assets/a5108412-b3db-4daa-b576-72608136a661" />

Encontramos una Web Shell o Command Injection mediante Web Shell. La shell es de www-data. Comprobamos que es una shell y que permite ejecutar comandos

<img width="883" height="142" alt="imagen" src="https://github.com/user-attachments/assets/83e5e6c8-a005-4b69-bbbb-cec7ad67d256" />

<img width="1234" height="116" alt="imagen" src="https://github.com/user-attachments/assets/27260cfd-ae89-4e36-8a4f-c0edc53f1727" />

<img width="1593" height="205" alt="imagen" src="https://github.com/user-attachments/assets/b4173c1f-945c-437c-8f23-4da26c3644cf" />

Vemos que hay dos usuarios reales:
```bash
rick:x:1000:1000::/home/rick:/bin/bash
negan:x:1001:1001::/home/negan:/bin/bash
```

Con ellos podemos probar fuerza bruta quizás sobre ssh. Podemos ver que procesos hay corriendo y si algún usuario usa SSH:
```bash
?cmd=ps aux
```

<img width="1593" height="259" alt="imagen" src="https://github.com/user-attachments/assets/5314c3fe-c68d-4e80-88e7-eae9f8d0f7e0" />

Podemos ver como rick tiene sesiones SSH activas.
