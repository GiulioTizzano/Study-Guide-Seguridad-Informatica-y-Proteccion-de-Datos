# PRACTICA PKI PARA HTTPS

En esta práctica se aborda el protocolo HTTPS (Hypertext Transfer Protocol over Secure Socket Layer), que es la base del servicio WWW seguro. También se trata la certificación X.509 y las infraestructuras PKI. Cubriremos los siguientes objetivos:

- Descripción de las conexiones HTTPS
- Presentación de la librería OpenSSL
- Configuración del modulo SSL/TLS de Apache. Servidores virtuales.
- Definición de una CA (Autoridad de Certificación) y gestión de certificados
- Autentificación de clientes Web mediante certificados

Usaremos una máquina Linux con distribución Ubuntu para construir la PKI y configurar servidores virtuales HTTPS. Se recomienda usar tanto la distribución Rocky como la Ubuntu que para la configuración de los servidores HTTPS. A la hora de realizar las pruebas HTTPS usaremos nuestro
propio equipo de usuario, o cualquier otro que cuente con interface gráfico y navegador web

![alt text](./imgs/ubuntu.png)
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
---
---

### 2. Instalacion de Apache y de SSL
---

En la **Ubuntu** instalamos ambos **Apache y SSL**
```bash
# Apache
sudo apt update
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2 # Comprobar que funciona con un curl o en la maquina windows

# SSL
sudo apt install mod_ssl -y # Se crea automaticamente un certificado autofirmado en /etc/pki/tls/certs/localhost.cert y la clave en **/etc/pki/tls/private/localhost.key
```

Verificamos que está levantado el puerto 443 (hay que reiniciar el servicio de httpd si nos se descargan a la vez):
```bash
sudo systemctl restart httpd
sudo ss -tlnp | grep httpd
```
---

En la **Rocky** descargamos **Apache**:
```bash
sudo dnf update
sudo dnf install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
```

---
---

### 3. Ubicacion de openssl.cnf y creacion de la CA
