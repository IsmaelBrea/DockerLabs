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

<img width="787" height="108" alt="imagen" src="https://github.com/user-attachments/assets/b8d31ad4-3460-40cd-84ec-bc0139b51463" />

```
                          KALI
                  eth0: 192.168.231.129/24
                           │
                           │
                    ┌──────┴──────┐
                    │ Docker Host │
                    │             │
                    │ br-...      │
                    │ 172.19.0.1  │
                    └──────┬──────┘
                           │
                 dockernetwork
                 172.19.0.0/16
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │    TRUST    │  │    UPLOAD   │  │  INCLUSION  │
   │ 172.19.0.2  │  │ 172.19.0.3  │  │ 172.19.0.4  │
   └─────────────┘  └─────────────┘  └─────────────┘
```
















