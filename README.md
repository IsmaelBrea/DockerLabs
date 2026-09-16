# DockerLabs

Este repositorio contiene la **resolución de distintas máquinas de DockerLabs**, una plataforma enfocada en el **aprendizaje de ciberseguridad práctica** mediante entornos dockerizados.

El objetivo es:

- Documentar paso a paso la resolución de cada máquina.  
- Explicar las técnicas, herramientas y comandos utilizados.  
- Servir como material de consulta y repaso.

**Máquinas:**
- Trust: Enumeración web → fuerza bruta de SSH → acceso como usuario → escalada de privilegios abusando de sudo con Vim.
- Injection: SQL Injection para obtener credenciales → acceso por SSH → escalada de privilegios mediante un binario con SUID.
- Little pivoting: Máquina enfocada específicamente en pivoting, donde debes comprometer una máquina y utilizarla como puente para alcanzar otra máquina/red interna que no es accesible directamente desde Kali.
- Chocolate Fire: Enumeración de servicios → explotación de una vulnerabilidad en Openfire para obtener una reverse shell como root.
- Domain: Enumeración SMB → fuerza bruta → acceso como usuario → explotación de recurso web → escalada de privilegios con SUID.
- Find_Your_Style: explotación de Drupal → acceso como www-data → escalada de privs → abuso de binarios para acceder a archivos de root.
- Walking Dead: Enumeración de servicios → identificación de una vulnerabilidad en Webmin → explotación para obtener acceso al sistema → escalada de privilegios hasta root.
- Walking CMS: Enumeración web → identificación y explotación de una vulnerabilidad en un CMS → obtención de acceso al servidor → escalada de privilegios hasta root.
- Winterfell: Enumeración web y SMB -> escalada de privilegios mediante movimiento entre usuarios.
  
