# Práctica 1 VPN:

**1. Se nos pide crear el escenario propuesto y configurar las direcciones IP de todos los interfaces
    y activar el forwarding en el gateway (para poder reenviar los datagramas). Establecer el router como gateway por defecto en todos los equipos
    y comprobar la conectividad.**

**- Gateway:**
```
# Ver los nombres de los interfaces reales de la máquina:
nmcli con show

# Cambiar el nombre a un interfaz (solo si hace falta):
nmcli con mod "Conexión cableada 1" connection.id eth1


# eth0 -> Red externa
nmcli con mod eth0 ipv4.addresses 172.22.0.10/24 ipv4.method manual
# eth1 -> Red interna
nmcli con mod eth1 ipv4.addresses 40.40.40.10/24 ipv4.method manual
nmcli con mod eth0 ipv4.gateway ""   # No gateway interno
nmcli con mod eth1 ipv4.gateway 40.40.40.1
(no necesario) nmcli con mod eth1 ipv4.dns 8.8.8.8
nmcli con up eth0
nmcli con up eth1

```
**- Servidor VPN:**
```
nmcli con mod eth0 ipv4.addresses 172.22.0.80/24 ipv4.method manual
nmcli con mod eth0 ipv4.gateway 172.22.0.10
(no necesario) nmcli con mod eth0 ipv4.dns 8.8.8.8
nmcli con up eth0
```

**- PC1 (cliente1):**
```
nmcli con mod eth0 ipv4.addresses 40.40.40.30/24 ipv4.method manual
nmcli con mod eth0 ipv4.gateway 40.40.40.10
(No hace falta) nmcli con mod eth0 ipv4.dns 8.8.8.8
nmcli con up eth0

```

**- PC2 (cliente2):**
```
nmcli con mod eth0 ipv4.addresses 172.22.0.50/24 ipv4.method manual
nmcli con mod eth0 ipv4.gateway 172.22.0.10
(No es necesario) nmcli con mod eth0 ipv4.dns 8.8.8.8
nmcli con up eth0
```
# **Activar el IP forwarding (solo en el gateway):**
Ir al gateway y aplicar los siguientes cambios:

Ir al fichero:
```
sudo vi /etc/sysctl.conf
```

Agregar o descomentar la línea:
```
net.ipv4.ip_forward = 1
```

Aplicar los cambios:
```
sudo sysctl -p
```

Verificar que se han realizado los cambios:
```
sysctl net.ipv4.ip_forward (se debe mostrar esto --> net.ipv4.ip_forward = 1
)
```
**Posteriormente comprobar la conectividad entre los dispositivos realizando los PINGs necesarios**

Para poder instalar openssh y apache en Rocky podemos ejecutar los siguientes comandos (por si acaso):

```
sudo dnf install -y httpd openssh-server
sudo systemctl enable --now httpd
sudo systemctl enable --now sshd
```
Verificar que apache funciona:
```
curl http://localhost
```

**2. Crear una Autoridad de Certificación en el servidor y emitir los elementos necesarios para la configuración (valores DH, claves/certificados para el servidor y el cliente.**

Para poder realizar este apartado bien, vamos a instalar las herramientas correspondientes necesarias (ejecutar en el servidor):

```
Activar el repositorio EPEL (requerido por easy-rsa):
sudo dnf install -y epel-release

Instalar OpenVPN y EasyRSA:
sudo dnf install -y openvpn easy-rsa

Herramientas adicionales (para diagnóstico y red):
sudo dnf install -y iproute iptables tcpdump net-tools bind-utils
```

Luego, pasamos a crear un entorno de trabajo para el PKI:

```
sudo mkdir -p /etc/openvpn/easy-rsa
sudo cp -r /usr/share/easy-rsa/3/* /etc/openvpn/easy-rsa/
sudo chown -R $(whoami) /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
```
Posteriormente, inicializamos la estructura de la PKI (creará el directorio pki/):

```
./easyrsa init-pki
```

Ahora pasamos a crear la autoridad de Certificación (CA):

Creamos la CA, que será la encargada de firmar los certificados del servidor y los clientes:

```
./easyrsa build-ca
```
Introducimos una frase de paso (que queramos), y asignamos un Common Name (CN) representativo (yo he puesto RedVPN-CA, como ejemplo).


Ahora, procedemos con la generación de claves y certificados:

Clave y certificado del servidor:
```
./easyrsa build-server-full servidor nopass
```

Clave y certificado del cliente:
```
./easyrsa build-client-full cliente nopass
```

Parámetros Diffie-Hellman:
```
./easyrsa gen-dh
```
-------------------------------------------------------------------------
Archivos Obtenidos Descripción:

pki/issued/servidor.crt	Certificado público del servidor
pki/private/servidor.key	Clave privada del servidor
pki/issued/cliente.crt	Certificado público del cliente
pki/private/cliente.key	Clave privada del cliente
pki/dh.pem	Parámetros Diffie-Hellman
pki/ca.crt	Certificado raíz de la CA
pki/private/ca.key	Clave privada de la CA

-------------------------------------------------------------------------

Generación de la clave TLS compartida (opcional, pero altamente recomendada):
```
cd /etc/openvpn
sudo openvpn --genkey --secret ta.key
```

Archivo que se ha tenido que crear (puede saltar un aviso de que está deprecado, pero sigue funcionando):
```
/etc/openvpn/ta.key
```

Ahora toca organizar los ficheros creados y pasarlos a la carpeta /etc/openvpn/server para que se quede todo bien guardado:
```
sudo cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/issued/servidor.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/private/servidor.key /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/server/
sudo cp /etc/openvpn/ta.key /etc/openvpn/server/

```

Comprobación final:
```
ls /etc/openvpn/server

Lo que deberia verse:
ca.crt  dh.pem  servidor.crt  servidor.key  ta.key
```








