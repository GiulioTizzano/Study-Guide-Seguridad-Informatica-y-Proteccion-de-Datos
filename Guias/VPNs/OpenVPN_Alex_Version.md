Redes Privadas Virtuales (Guía Completa)

# Guía Completa: Redes Privadas Virtuales (OpenVPN y WireGuard)

Esta guía cubre la instalación y configuración de un escenario VPN completo, incluyendo configuración de servidor, cliente, gateway, routing, modo bridge y la alternativa WireGuard.

---

## 1. Preparación del Entorno (Parte A)

### Instalación de Paquetes

**Ubuntu (Gateway, Servidor, Cliente):**
```bash
sudo apt update
sudo apt install -y openvpn easy-rsa
# Para la parte opcional de bridge:
sudo apt install -y bridge-utils
```

**Rocky Linux / RHEL:**

``` Bash
sudo dnf install -y epel-release
sudo dnf update -y
sudo dnf install -y openvpn easy-rsa bridge-utils
```

### Configuración de Red (Gateway)

El Gateway debe tener dos interfaces: una externa (hacia PC1) y una interna (hacia Servidor/PC2). Se debe habilitar el **IP Forwarding** en el Gateway y en el Servidor VPN.

``` Bash
# Habilitar forwarding temporalmente
sudo sysctl -w net.ipv4.ip_forward=1

# Para hacerlo persistente editar /etc/sysctl.conf y añadir:
# net.ipv4.ip_forward=1
# Luego aplicar:
sudo sysctl -p
```

---

## 2. Crear la CA y Certificados (Parte B)

Realizar estos pasos en el **Servidor VPN**.

### Inicializar la PKI

``` Bash
cd ~
# Inicializar infraestructura de clave pública (crea directorio pki)
sudo /usr/share/easy-rsa/easyrsa init-pki

# Crear la Autoridad de Certificación (CA) sin contraseña
sudo /usr/share/easy-rsa/easyrsa build-ca nopass
```

### Generar Claves y Certificados

``` Bash
# Generar parámetros Diffie-Hellman
sudo /usr/share/easy-rsa/easyrsa gen-dh

# Generar certificado del SERVIDOR
sudo /usr/share/easy-rsa/easyrsa build-server-full servidor nopass

# Generar certificado del CLIENTE
sudo /usr/share/easy-rsa/easyrsa build-client-full cliente-01 nopass

# Generar clave HMAC (seguridad extra contra ataques DoS)
sudo openvpn --genkey secret /etc/openvpn/ta.key
```

### Organizar Archivos en el Servidor

``` Bash
# Copiar archivos necesarios al directorio de configuración
sudo cp /usr/share/easy-rsa/pki/ca.crt /etc/openvpn/server/
sudo cp /usr/share/easy-rsa/pki/issued/servidor.crt /etc/openvpn/server/
sudo cp /usr/share/easy-rsa/pki/private/servidor.key /etc/openvpn/server/
sudo cp /usr/share/easy-rsa/pki/dh.pem /etc/openvpn/server/
sudo cp /etc/openvpn/ta.key /etc/openvpn/server/

# Ajustar permisos (seguridad)
sudo chmod 600 /etc/openvpn/server/servidor.key
sudo chmod 600 /etc/openvpn/server/ta.key
```

---

## 3. Configuración del Servidor OpenVPN (Parte C)

Editar archivo: `/etc/openvpn/server/servidor.conf`

``` Bash
port 1194
proto udp
dev tun

# Rutas de certificados
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/servidor.crt
key /etc/openvpn/server/servidor.key
dh /etc/openvpn/server/dh.pem

# Configuración IP VPN
server 172.16.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Empujar ruta de la LAN interna (para que el cliente vea a PC2)
push "route 172.22.0.0 255.255.255.0"

keepalive 10 120
tls-auth /etc/openvpn/server/ta.key 0
cipher AES-256-CBC
persist-key
persist-tun
verb 4
```

**Arrancar servicio:**

``` Bash
sudo systemctl enable openvpn-server@servidor
sudo systemctl start openvpn-server@servidor
```

---

## 4. Configuración del Gateway (Parte D y E)

El Gateway actúa como router de borde.

### Reglas IPTables (NAT)

``` Bash
# 1. Port Forwarding (DNAT): Tráfico al puerto 1194 externo va al servidor interno
sudo iptables -t nat -A PREROUTING -i eth1 -p udp --dport 1194 -j DNAT --to-destination 172.22.0.80:1194

# 2. Permitir paso de paquetes (Forwarding)
sudo iptables -A FORWARD -p udp -d 172.22.0.80 --dport 1194 -j ACCEPT

# 3. Enmascaramiento (Salida a internet de la red interna)
sudo iptables -t nat -A POSTROUTING -s 172.22.0.0/24 -o eth1 -j MASQUERADE
```

---

## 5. Configuración del Cliente (Parte F y G)

### Transferir archivos al Cliente (PC1)

Copiar desde el servidor los siguientes archivos a `/etc/openvpn/client/` en PC1 (usando SCP o USB):

- `ca.crt`
- `cliente-01.crt`
- `cliente-01.key`
- `ta.key`

Ejemplo:

En el servidor:
```Bash
sudo chown -R root:root /etc/openvpn/server
sudo cp /usr/share/easy-rsa/pki/ca.crt /etc/openvpn/server/
sudo cp /usr/share/easy-rsa/pki/issued/servidor.crt /etc/openvpn/server/
sudo cp /usr/share/easy-rsa/pki/private/servidor.key /etc/openvpn/server/
sudo cp /usr/share/easy-rsa/pki/dh.pem /etc/openvpn/server/
sudo cp /etc/openvpn/ta.key /etc/openvpn/server/
```

*Permisos:*
```Bash
sudo chmod 600 /etc/openvpn/server/servidor.key 
sudo chmod 600 /etc/openvpn/server/ta.key 
sudo chown root:root /etc/openvpn/server/*
```

En el GW:

```Bash
# Redirigir el puerto 2222 del GW al puerto 22 del Servidor VPN (172.22.0.80)
sudo iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 2222 -j DNAT --to-destination 172.22.0.80:22

sudo iptables -A FORWARD -p tcp -d 172.22.0.80 --dport 22 -j ACCEPT
```

En el cliente:

```Bash
# La opción -P 2222 indica el puerto mapeado en el Gateway
sudo scp -P 2222 root@40.40.40.10:/usr/share/easy-rsa/pki/ca.crt /etc/openvpn/client/
sudo scp -P 2222 root@40.40.40.10:/usr/share/easy-rsa/pki/issued/cliente-01.crt /etc/openvpn/client/
sudo scp -P 2222 root@40.40.40.10:/usr/share/easy-rsa/pki/private/cliente-01.key /etc/openvpn/client/
sudo scp -P 2222 root@40.40.40.10:/etc/openvpn/ta.key /etc/openvpn/client/
```

En caso de necesitar hacer un ProxyJump:
```Bash
scp -o ProxyJump=root@40.40.40.10 root@172.22.0.80:/usr/share/easyrsa/pki/ca.crt /etc/openvpn/client/ 

scp -o ProxyJump=root@40.40.40.10 root@172.22.0.80:/usr/share/easyrsa/pki/issued/cliente-01.crt /etc/openvpn/client/

scp -o ProxyJump=root@40.40.40.10 root@172.22.0.80:/usr/share/easyrsa/pki/private/cliente-01.key /etc/openvpn/client/

scp -o ProxyJump=root@40.40.40.10 root@172.22.0.80:/etc/openvpn/ta.key /etc/openvpn/client/
```

*Permisos:*
```Bash
sudo chmod 600 /etc/openvpn/client/cliente-01.key
sudo chown root:root /etc/openvpn/client/*
```
### Archivo de configuración Cliente

Editar: `/etc/openvpn/client/cliente.conf`

```Bash
client
dev tun
proto udp

# IP PÚBLICA del Gateway
remote 40.40.40.10 1194

resolv-retry infinite
nobind

# Certificados
ca /etc/openvpn/client/ca.crt
cert /etc/openvpn/client/cliente-01.crt
key /etc/openvpn/client/cliente-01.key
remote-cert-tls server
tls-auth /etc/openvpn/client/ta.key 1

cipher AES-256-CBC
verb 3
```

**Conectar:**

```Bash
sudo openvpn --config /etc/openvpn/client/cliente.conf
```

---

## 6. Solución de Routing (Parte H e I)

Si PC1 conecta pero no ve a PC2, es un problema de rutas de retorno.

**Solución en el Gateway (Recomendada):** Añadir ruta para saber que la red VPN (172.16.0.0) está detrás del Servidor VPN.

```Bash
# Ruta estática
sudo ip route add 172.16.0.0/24 via 172.22.0.80 dev eth0

# Permitir tráfico entre LAN y VPN
sudo iptables -A FORWARD -s 172.22.0.0/24 -d 172.16.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 172.16.0.0/24 -d 172.22.0.0/24 -j ACCEPT
```

---
## 7. Modo Bridged (Opcional - Parte J)

Permite asignar IPs de la LAN física a los clientes VPN.
### Scripts de Bridge (Servidor)

Crear `/etc/openvpn/bridge-up.sh`:

```sh
#!/bin/bash
BR="$1"
PHY="$2" # Interfaz física (ej. eth0)
TAP="$3" # Interfaz virtual (tap0)

# Obtener IP actual y crear bridge
IP_INFO=$(ip -4 addr show dev $PHY | awk '/inet /{print $2; exit}')
ip link add name $BR type bridge 2>/dev/null || true
ip link set dev $PHY down
ip link set dev $TAP down
ip link set dev $PHY master $BR
ip link set dev $TAP master $BR

# Asignar IP al bridge y levantar
if [ -n "$IP_INFO" ]; then
  ip addr flush dev $PHY
  ip addr add $IP_INFO dev $BR
fi
ip link set dev $BR up
ip link set dev $PHY up
ip link set dev $TAP up
```

### Configuración OpenVPN (Cambios)

1. Cambiar `dev tun` por `dev tap` en servidor y cliente.
2. Comentar `server 172.16.0.0...`.
3. Usar `server-bridge` o DHCP.
4. Reiniciar servicio.

---

## 8. WireGuard (Opcional - Parte K)

### Instalación y Claves

```Bash
sudo apt install wireguard
# Generar claves (Servidor y Cliente)
wg genkey | tee private.key | wg pubkey > public.key
```

### Configuración Servidor (/etc/wireguard/wg0.conf)

Ini, TOML

```Java
[Interface]
PrivateKey = <Server_Private_Key>
Address = 10.8.0.1/24
ListenPort = 51820

[Peer]
# PC1
PublicKey = <Client_Public_Key>
AllowedIPs = 10.8.0.2/32
```

### Configuración Cliente (/etc/wireguard/wg0.conf)

Ini, TOML

```Python
[Interface]
PrivateKey = <Client_Private_Key>
Address = 10.8.0.2/24

[Peer]
# Servidor
PublicKey = <Server_Public_Key>
Endpoint = 40.40.40.10:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### Firewall (Gateway)

Redirigir puerto UDP 51820 igual que se hizo con OpenVPN.

```Bash
sudo iptables -t nat -A PREROUTING -i eth1 -p udp --dport 51820 -j DNAT --to-destination 172.22.0.80:51820
```

---
## Ejemplos para identificar los certificados AI RESPONSE:

### 1. El Certificado de la CA (`ca.crt`)

Es el certificado raíz. Se identifica porque el emisor (**Issuer**) y el sujeto (**Subject**) son el mismo (está autofirmado).

- **Cómo se ve el archivo:** Empieza por `-----BEGIN CERTIFICATE-----`.
- **Cómo identificarlo por comando:**
```Bash
 openssl x509 -in ca.crt -text -noout
 ```
- **En qué fijarte:** Busca la línea `Subject: CN=Easy-RSA CA`. Si el `Issuer` y el `Subject` coinciden, es la CA.

---

### 2. El Certificado del Cliente (`cliente-01.crt`)

Es la "tarjeta de identidad" del cliente. Contiene su clave pública y ha sido firmado por la CA.

- **Cómo se ve el archivo:** Empieza por `-----BEGIN CERTIFICATE-----`.
- **Cómo identificarlo por comando:*

```Bash
openssl x509 -in cliente-01.crt -text -noout
```

- **En qué fijarte:** * **Subject:** Dirá algo como `CN=cliente-01`.
    - **Issuer:** Dirá `CN=Easy-RSA CA` (esto indica quién lo firmó).
    - **X509v3 Extended Key Usage:** Suele decir `TLS Web Client Authentication`.

---

### 3. La Clave Privada (`cliente-01.key`)

Es la parte más sensible. **Nunca debe compartirse**. A diferencia de los certificados, este archivo no es "legible" como texto de información, sino que es un bloque de datos cifrados o codificados.

- **Cómo se ve el archivo:**
```Python
-----BEGIN PRIVATE KEY-----
   MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDPr9...
   ... (muchas líneas de caracteres aleatorios) ...
-----END PRIVATE KEY-----
```
- **Nota:** Si ves la palabra `PRIVATE KEY`, sabes inmediatamente que es una clave y no un certificado.
---
### 4. La Clave HMAC (`ta.key`)

Esta clave es distinta a todas las anteriores. No sigue el formato de certificados X.509, sino que es una clave simétrica simple para OpenVPN.

- **Cómo se ve el archivo:** Es muy fácil de reconocer porque tiene un encabezado específico de OpenVPN:

```Python
   #
   # 2048 bit OpenVPN static key
   #
   -----BEGIN OpenVPN Static key V1-----
   e9d407336f32819d6594f8379435868e
   ... (letras y números hexadecimales) ...
   -----END OpenVPN Static key V1-----
   ```