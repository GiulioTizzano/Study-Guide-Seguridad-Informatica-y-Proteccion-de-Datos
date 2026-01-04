**1. Se nos pide crear el escenario propuesto y configurar las direcciones IP de todos los interfaces
    y activar el forwarding en el gateway (para poder reenviar los datagramas). Establecer el router como gateway por defecto en todos los equipos
    y comprobar la conectividad.**

**- Gateway:**
```
# Editar el archivo de configuración del direccionamiento:
sudo vi /etc/netplan/50-cloud-init-yaml

# Copiar y pegar esto, pero obviamente cambiando el direccionamiento según el enunciado:

network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 40.40.40.10/24
    ens37:
      addresses:
        - 172.22.0.10/24
      nameservers:
        addresses: [8.8.8.8]
```
Luego, ejecutar para ver si se configuran correctamente los cambios:
```
sudo netplan apply
```

**- Servidor VPN:**
```
# Ahora volvemos a editar el mismo archivo pero para el servidor VPN y metemos su direccionamiento correspondiente:

network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.22.0.80/24

      routes:
      - to: default
        via: 172.22.0.10
      nameservers:
        addresses: [8.8.8.8]
```
Luego, volver a ejecutar el siguiente comando para guardar los cambios:
```
sudo netplan apply
```

**- PC1:**

```
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 40.40.40.30/24
      routes:
      - to: default
        via: 40.40.40.10
      nameservers:
        addresses: [8.8.8.8]
```
Luego, volver a ejecutar el siguiente comando para guardar los cambios:
```
sudo netplan apply
```

**- PC2:**
```
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.22.0.50/24
      routes:
      - to: default
        via: 172.22.0.10
      nameservers:
        addresses: [8.8.8.8]
```
Luego, volver a ejecutar el siguiente comando para guardar los cambios:
```
sudo netplan apply
```

**Activar el IP forwarding (solo en el gateway):**
Para activar el IP forwarding en el gateway, vamos al siguiente fichero:
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

Para poder instalar openssh y apache en Ubuntu podemos ejecutar los siguientes comandos (por si acaso):
```
sudo apt update && upgrade
sudo apt install -y openssh-server apache2
```

Verificar que apache funciona:
```
curl http://localhost
```

**2. Crear una Autoridad de Certificación en el servidor y emitir los elementos necesarios para la configuración (valores DH, claves/certificados para el servidor y el cliente.**

Para poder realizar este apartado bien, vamos a instalar las herramientas correspondientes necesarias (ejecutar en el servidor):

```
Instalar OpenVPN y EasyRSA:
sudo apt install -y openvpn easy-rsa
```

Ahora, preparamos el directorio Easy-RSA:

```
sudo mkdir -p /etc/openvpn/easy-rsa
sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
sudo chown -R $USER:$USER /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
```

Inicializar PKI y crear CA:

```
./easyrsa init-pki
./easyrsa build-ca
```

Ahora procedemos a crear el servidor, cliente, DH y CRL:
```
./easyrsa build-server-full servidor nopass
./easyrsa build-client-full cliente nopass
./easyrsa gen-dh
./easyrsa gen-crl
```

Clave TLS compartida (ta.key):
```
sudo openvpn --genkey --secret /etc/openvpn/ta.key
```

Organizar ficheros en /etc/openvpn/server
```
sudo mkdir -p /etc/openvpn/server

sudo cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/issued/servidor.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/private/servidor.key /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/crl.pem /etc/openvpn/server/
sudo cp /etc/openvpn/ta.key /etc/openvpn/server/

# (si quieres dejar los del cliente aquí temporalmente como en tu guía)
sudo cp /etc/openvpn/easy-rsa/pki/issued/cliente.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/private/cliente.key /etc/openvpn/server/
```

Comprobación:
```
ls -l /etc/openvpn/server
```

```
Deberías ver:
ca.crt crl.pem dh.pem servidor.crt servidor.key ta.key cliente.crt cliente.key
```

**3. Configurar el servidor OpenVPN en modo TUN. Usaremos UDP como protocolo de transporte (puerto 1194), configurar su arranque automático e iniciar el servicio. Habilitar el forwarding del servidor para que pueda encaminar el tráfico entre sus interfaces.**

Creamos el fichero **/etc/openvpn/server/server.conf**:

```
sudo vi /etc/openvpn/server/servidor.conf
```

Configuración dentro del fichero **server.conf**:
```

# ================================
# CONFIGURACIÓN DEL SERVIDOR OPENVPN (MODO TUN - UDP)
# ================================

# Puerto y protocolo de transporte
port 1194
proto udp

# Dispositivo virtual de nivel 3 (túnel IP)
dev tun

# Rutas a los certificados y claves
ca ./ca.crt
cert ./servidor.crt
key ./servidor.key
dh ./dh.pem
crl-verify ./crl.pem

# Clave TLS compartida para canal de control (opcional pero recomendada)
tls-auth ./ta.key 0

# Red virtual que OpenVPN asignará a los clientes
server 172.16.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt

# Anunciar a los clientes la red interna detrás del servidor (LAN real)
push "route 172.22.0.0 255.255.255.0"

# Parámetros de sesión: keepalive (ping/timeout)
keepalive 10 120

# Cifrado simétrico del túnel
data-ciphers AES-256-CBC

# Mantener claves y túnel tras reinicios del servicio
persist-key
persist-tun

# Nivel de verbosidad del log (4 = detallado)
verb 4
```

Luego, tenemos que activar el **FORWARDING** también en el servidor, para ello manipulamos el fichero que se encuentra aquí:

```
sudo vi /etc/sysctl.conf

y agregar o descomentar la siguiente línea:

net.ipv4.ip_forward = 1
```

Aplicamos los cambios:
```
sudo sysctl -p
sysctl net.ipv4.ip_forward

Debería verse como resultado:
net.ipv4.ip_forward = 1
```

Arrancamos el serivico de OpenVPN en el lado del servidor:
```
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server
```

Comprobación del estado:
```
sudo systemctl status openvpn-server@server

Si salta algún error, ejecutar el siguiente comando:

sudo journalctl -u openvpn-server@server (-n 20)
```

Comprobaciones de funcionamiento:

Ver interfaz TUN creada
```
ip addr show
ip a
ip a show tun0
```
Ver rutas activas
```
ip route
```

**4. Nos piden configurar en el gateway un port-forwarding para que el servidor interno (LAN) sea capaz de recibir el tráfico recibido en el puerto udp/1194 de su interfaz externo (gateway) (es necesaria una regla iptables NAT en PREROUTING).**

Para realizar la configuración del port-forwarding (PREROUTING) debemos primero de conocer cuales son los interfaces asociados al gateway. Para ello, ejecutamos lo siguiente:

```
ip a
```

Con esto, deberíamos de ver los interfaces con sus IPs asociadas. Nosotros, vamos a aplicar la regla de PREROUTING al interfaz que dé directamente CONECTIVIDAD hacia el exterior (en mi caso es el ens33 de mi gateway). Por tanto, ejecutamos el siguiente comando para aplicar la regla según lo que pide el enunciado:

```
sudo iptables -t nat -A PREROUTING -p udp --dport 1194 -i eth1 -j DNAT --to-destination 172.22.0.80:1194
```

En este caso la IP **172.22.0.80** es la dirección IP de la interfaz de mi servidor interno.

También, para comprobar que la regla se haya aplicado correctamente, deberíamos ejecutar lo siguiente desde el GATEWAY:

```
sudo iptables -t nat -L PREROUTING -n -v
```

Dentro de la Ubuntu server, las reglas de iptables no se guardan automáticamente y para que sean persistentes tras reiniciar el gateway hace falta ejecutar el siguiente comando (dentro del GATEWAY):

```
sudo apt update
sudo apt install iptables-persistent


sudo netfilter-persistent save

sudo systemctl status netfilter-persistent

```
La próxima vez que arranquemos la máquina la regla debería de haberse guardado.

**5. Configurar en el Gateway el enmascaramiento para todo el tráfico saliente. También hay que comprobar que se siguen pudiendo alcanzar todos los recursos externos desde la red interna (PC2 a PC1). Verificar que se realiza el enmascaramiento usando tcpdump.**

Ejecutamos el siguiente comando para enmascarar el tráfico (lo ejecutamos en el GATEWAY):

```
sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
```

Comprobamos que se haya completado correctamente la regla ejecutando el siguiente comando:

```
sudo iptables -t nat -L POSTROUTING -n -v
```

Para guardar la configuración (darle persistencia), debemos de ejecutar los siguientes comandos:

```
sudo apt update
sudo apt install iptables-persistent


sudo netfilter-persistent save

sudo systemctl status netfilter-persistent

```
La próxima vez que arranquemos la máquina la regla debería de haberse guardado.

Procedemos a realizar ping desde la máquina PC2 a la PC1 (externa --> interna).

**6. Configurar el cliente OpenVPN en el PC1 (client.conf) para conectar al servidor (a través de la IP del GATEWAY). Para completar esta tarea es necesario copiar los archivos necesarios obtenidos en el apartado 2.**

```
sudo dnf install -y openvpn
```

Luego, creamos el directorio de configuración del cliente:

```
sudo mkdir -p /etc/openvpn/client
cd /etc/openvpn/client
```

Ahora copiamos los archivos necesarios para el cliente desde el servidor VPN. Eso incluye el ca.crt, cliente.crt, cliente.key y el ta.key:

```
scp root@172.22.0.80:/etc/openvpn/server/ca.crt .
scp root@172.22.0.80:/etc/openvpn/easy-rsa/pki/issued/cliente.crt .
scp root@172.22.0.80:/etc/openvpn/easy-rsa/pki/private/cliente.key .
scp root@172.22.0.80:/etc/openvpn/server/ta.key .
```

Tras copiar todos los ficheros correspondientes a nuestra máquina cliente, deberíamos ver lo siguiente:

```
ca.crt
cliente.crt
cliente.key
ta.key
```

Ahora creamos el archivo de configuración del cliente **client.conf** y le metemos la siguiente estructura al archivo:


```
sudo vi /etc/openvpn/client/client.conf


# ================================
# CLIENTE OPENVPN - MODO TUN (UDP)
# ================================

# Dirección y puerto del servidor VPN:
client
dev tun
proto udp

# IP externa del gateway:
remote 40.40.40.10 1194

# Certificados y claves:
ca ./ca.crt
cert ./cliente.crt
key ./cliente.key

# Clave TLS compartida (coincide con servidor):
tls-auth ./ta.key 1 (en mi caso la llave se llama ta-key, ajustar según necesidad)
remote-cert-tls server

# Algoritmo de cifrado:
data-ciphers AES-256-CBC

# Mantener la conexión
resolv-retry infinite
persist-key
persist-tun
```

Una vez configurado, le damos a iniciar el servicio, habilitamos el servicio para que arranque en cuanto arranque la máquina y comprobamos su estado:

```
sudo systemctl start openvpn-client@client
sudo systemctl enable openvpn-client@client
sudo systemctl status openvpn-client@client

```

**7. Iniciar el cliente y comprobar que se conecta al servidor. Verificar la configuración que se establece en el interfaz virtual TUN del cliente, tabla de encaminamiento y que es posible alcanzar desde PC1 los recursos ofrecidos por el servidor a través de la VPN. A su vez, desactivar el router por defecto en PC1 y comprobar que sigue siendo posible acceder al servidor**

Iniciar el cliente OpenVPN:
```
sudo systemctl start openvpn-client@client
sudo systemctl status openvpn-client@client
```

Comprobar el interfaz virtual tun:

```
ip a show tun0 (se debería ver la configuración punto a punto de las direcciones IP)
```

Verificamos la tabla de encaminamiento en PC1:
```
ip route
```

Comprobar conectividad hacia los recursos del servidor:

```
ping 172.16.0.1 (debemos obtener respuesta del PING)
ping 172.22.0.80 (debemos obtener respuesta del PING)
```

Verificar que el tráfico se transporta por tun0 y cifrado sobre ens37:

En el lado servidor:
```
sudo tcmdump -i tun0 -n 
```
En PC1:
```
ping 172.22.0.80
```

En la captura veremos ICMP normales porque tun0 ya contiene el tráfico descifrado

En cambio, cuando miramos al interfaz del gateway (ens37) que conecta a la red interna, veremos que las capturas deben mostrar paquetes UDP cifrados. Dado que, PC1 encapsula su tráfico dentro de OpenVPN, el gateway reenvía el tráfico cifrado y el servidor descifra en tun0:

En el gateway:
```
sudo tcpdump -i eth0 udp port 1194 -n
```


En PC1:
```
ping 172.22.0.80
```

Ahora desactivamos el router por defecto en PC1 y comprobamos el acceso al servidor, en el caso de la ubuntu server simplemente entrando en el netplan y eliminando el router default to/via:
```
# Guardar los cambios de borrar el default gateway del netplan:

sudo netplan apply
```

Luego, lanzamos ping al servidor:
ping 172.22.0.80


Debería de funcionar la conectividad sin necesidad de la ruta por defecto, porque las comunicaciones pasan por el canal del túnel.


**8. Comprobar si es posible alcanzar el equipo PC2 desde PC1 a través de la conexión VPN.**

Prueba desde PC1 a PC2:
```
ping 172.22.0.50 

Lo normal es que no responda, pero no porque PC2 no reciba los paquetes, sino porque PC2 no sabe como volver a PC1 (no conoce como llegar a PC1).

Si analizamos las capturas al hacer ping de PC1 --> PC2, veremos que a PC2 le llegan todos los datagramas:

sudo tcmdump -i eth0 -n
```

**9. Modificar la configuración de PC2 par que pueda comunicar con PC1 a través de la VPN. (También es posible configurar el gateway para el retorno de los paquetes a través de la VPN, sin modificar el encaminamiento de PC2) .**

Configuración de PC2 para poder comunicar con PC1:
```
sudo ip route add 172.16.0.0/24 via 172.22.0.80
```

O también, es posible configurar la devuelta de los paquetes en el propio Gateway (más cómodo en el caso hipotético de que hubiera más dispositivos en la red y de no querer configurar uno por uno los dispositivos):

```
sudo ip route add 172.16.0.0/24 via 172.22.0.80

```
Ambas opciones han de funcionar

FIN PRÁCTICA











