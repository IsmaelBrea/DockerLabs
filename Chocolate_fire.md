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













Hemos alcanzado el nivel de privilegios máximos en el sistema!



















