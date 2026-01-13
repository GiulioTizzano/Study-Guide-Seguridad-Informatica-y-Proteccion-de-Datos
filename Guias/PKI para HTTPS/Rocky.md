# PRACTICA PKI PARA HTTPS

En esta práctica se aborda el protocolo HTTPS (Hypertext Transfer Protocol over Secure Socket Layer), que es la base del servicio WWW seguro. También se trata la certificación X.509 y las infraestructuras PKI. Cubriremos los siguientes objetivos:

- Descripción de las conexiones HTTPS
- Presentación de la librería OpenSSL
- Configuración del modulo SSL/TLS de Apache. Servidores virtuales.
- Definición de una CA (Autoridad de Certificación) y gestión de certificados
- Autentificación de clientes Web mediante certificados

Usaremos una máquina Linux con distribución Rocky para construir la PKI y configurar servidores virtuales HTTPS. Se recomienda usar tanto la distribución Rocky como la Ubuntu que para la configuración de los servidores HTTPS. A la hora de realizar las pruebas HTTPS usaremos nuestro
propio equipo de usuario, o cualquier otro que cuente con interface gráfico y navegador web

![alt text](./imgs/rocky.png)
---

### 1. Comprobar fecha y hora de la máquina virtual. Si no está bien, actualizar la fecha y hora del sistema (importante porque si no fallará al emitir los certificados)
>**IMPORTANTE:** Esto es importante para la maquina que emite los certificados

Nos fijamos que la fecha y hora esten correctas:
```bash
date
timedatectl
```
Si la zona horaria no fuera correcta y además NTP no estuviera activo entonces deberíamos ejecutar la siguiente serie de comandos:

```bash
sudo timedatectl set-timezone Europe/Madrid # (poner la zona horaria)
timedatectl # (ver la información de la zona y fecha)
sudo dnf install –y chrony # (insalar el servicio chrony – cliente moderno de NTP)
sudo systemctl enable –now chronyd # (habilitar el servicio)
system status chronyd # (mirar el status del demonio)
chronyc tracking # (verificar sincronización)
sudo chronyc makestep # (Forzar la sincronización inmediata)
Timedatectl # (Verificar la información)
```

### 2. Instalacion de Apache y de SSL

