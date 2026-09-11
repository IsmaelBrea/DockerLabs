# Máquina Walking CMS 

Dificultad -> Fácil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

**Imprescindible tener instalado Docker**. Docker te permite ejecutar programas de forma aislada y portátil, como si cada programa tuviera su propio mini-ordenador dentro de tu PC.

## Despliegue del laboratorio
 Una vez descargada la máquina, debemos realizar los siguientes pasos para tener acceso a la máquina:
 
 En primer lugar veremos que tenemos descargado un zip con dos archivos un .sh y un .tar.


Para el despliegue de la máquina tendremos que ejecutar el siguiente comando:
```shell
sudo bash auto_deploy.sh walkingcms.tar
```

<img width="732" height="511" alt="imagen" src="https://github.com/user-attachments/assets/e7074790-c08b-4524-87be-5d701dc124d2" />


Realizamos un escaneo en la máquina para obtener servicios abiertos:
```bash
nmap -sCV -p- --open -min-rate 5000 172.17.0.2 -vvv
```

Obtenemos lo siguiente:
<img width="814" height="169" alt="imagen" src="https://github.com/user-attachments/assets/0fddbe54-d91e-4b13-b6b3-435e658443dc" />

Sólo tenemos una web corriendo en la máquina y viendo el título de la máquina podemos intuir que se trata de una web que utiliza un CMS. UN CMS (Content Management System) es un sistema de gestión de contenido, es decir, es una aplicación que permite crear, editar y administrar una página web sin tener que programarla completamente desde cero. Ejemplos de CMS conocidos son Wordpress, Joomla o Drupal. Suelen tener muchas vulnerablidades en sus plugins si las versiones no están actualizadas. En el reporte de nmap vemos que usa PHP y que corre un Apache 2 Debian.

Vamos a ver que hay en la web:


<img width="1784" height="822" alt="imagen" src="https://github.com/user-attachments/assets/29cf3ec8-5431-4cb3-863a-1145d4c050ff" />


Nos encontramos la plantilla de Apache. Nada que explotar en la web de momento. Lo ideal ahora es aplicar fuzzing para ver si encontramos alguna ruta que contenga algo interesante.

Tras hacer un fuzzing básico nos encontramos lo siguiente:
<img width="1911" height="484" alt="imagen" src="https://github.com/user-attachments/assets/962a2908-0ad0-4e9a-b607-8de6caae6326" />

Gobuster se queja porque la web responde 200 incluso cuando la página no existe, y además esas respuestas tienen siempre 10701 bytes. Por eso Gobuster no sabe distinguir una página real de una falsa. Con --exclude-length 10701 le dices: “ignora todas las respuestas que midan 10701 bytes”, y así te mostrará las rutas que tengan una respuesta diferente.
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt --exclude-length 10701
```
Ahora si que obtenemos respuesta y tal y como afirmábamos obtener una url con un dominio que apunta a Wordpress. La ruta es /wordpress.

Ahora si vemos una web:

<img width="1898" height="849" alt="imagen" src="https://github.com/user-attachments/assets/bd49e982-427c-4348-999d-8c824dd77f59" />


Cuando queremos explotar un wordpress tenemos varios vectores de enumeración. Lo primero sería mirar la versión del wordpress utilizado. En el html con Ctrl+U buscamos algo que sea "wordpress"

Más cosas que podemos hacer luego son: buscar algún dominio más pero con la ruta /wordpress, usar `wpscan` para enumerar la web sus plugins y sus usuarios entre otras cosas y también podríamos analizar la web si es necesario con herramientas como `nikto` o `whatweb`. 

Mirando el código html de la web vemos que redirige a muchas subrutas de /wordpress. Si buscamos wordpress también encontramos la versión exacta que está usando.

<img width="394" height="21" alt="imagen" src="https://github.com/user-attachments/assets/365bd026-c9fe-44f6-94cf-ff8438669b54" />

Vamos a seguir enumerando cosas. Ahora vamos a entrar en las rutas básicas que tienen los proyectos wordpress para ver si encontramos algo, por ejemplo:
`wordpress/admin.php`, `wordpress/wp-login.php` y `wordpress/xmlrpc.php`

De estas rutas funcionan todas menos la primera. El xmlrcp nos dice que el servidor solo acepta peticiones POST, por lo que vamos a poder hacer fuerza bruta masiva.

Por tanto ya sabemos que versión de Wordpress tenemos y que podemos fuzzear el apartado de login. 

Vamos a seguir enumerando la web, en esta caso con wpscan para obtener plugins y usuarios.

| Recordamos que en este caso la web devuelve 200 para casi cualquier URL, así que WPScan cree que existen plugins que realmente no existen. Prueba a decirle que ignore las respuestas con el mismo contenido usand:

```bash
wpscan --url http://172.17.0.2/wordpress/ --plugins-detection aggressive --exclude-content-based "10701"
```

Encontramos:
| Hallazgo                       | Importancia  | Qué significa                                                                                                   |
| ------------------------------ | ------------ | --------------------------------------------------------------------------------------------------------------- |
| **XML-RPC habilitado**         | Alta         | `xmlrpc.php` acepta peticiones POST. Puede permitir enumeración de usuarios, ataques de login/XML-RPC, etc.     |
| **`backup-db/`**               | **Muy alta** | Hay un directorio de backups accesible: `/wp-content/backup-db/`. **Este es el primer sitio que investigaría.** |
| **`mu-plugins/`**              | Alta         | Hay *Must-Use Plugins*. Pueden contener código que no aparece como plugin normal.                               |
| **WordPress 7.1**              | Media        | Sabemos exactamente la versión instalada.                                                                       |
| **Tema Twenty Twenty-Two 1.6** | Media        | Está desactualizado respecto a la versión detectada por WPScan.                                                 |
| **PHP 8.2.7**                  | Baja/Media   | Información útil para identificar el stack.                                                                     |
| **`readme.html`**              | Baja         | Revela información sobre WordPress.                                                                             |
| **WP-Cron externo**            | Baja         | `wp-cron.php` es accesible externamente.                                                                        |





