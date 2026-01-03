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
        via: 172.22.0.10/24
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
        via: 40.40.40.10/24
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
        via: 172.22.0.10/24
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

**3. Configurar el servidor OpenVPN en modo. Usaremos UDP como protocolo de transporte (puerto 1194), configurar su arranque automático e iniciar el servicio. Habilitar el forwarding del servidor para que pueda encaminar el tráfico entre sus interfaces.**


