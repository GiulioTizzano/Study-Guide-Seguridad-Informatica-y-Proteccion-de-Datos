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
---
---

### 2. Instalacion de Apache y de SSL
---

En la **Rocky** instalamos ambos **Apache y SSL**
```bash
# Apache
sudo dnf update
sudo dnf install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd # Comprobar que funciona con un curl o en la maquina windows

# SSL
sudo dnf install mod_ssl -y  # Se crea automaticamente un certificado autofirmado en /etc/pki/tls/certs/localhost.cert y la clave en **/etc/pki/tls/private/localhost.key
```

Verificamos que está levantado el puerto 443 (hay que reiniciar el servicio de httpd si nos se descargan a la vez):
```bash
sudo systemctl restart httpd
sudo ss -tlnp | grep httpd
```
---

En la **Ubuntu** descargamos **Apache**:
```bash
sudo apt update
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2
```
---
---

### 3. Creacion de la CA, creacion de claves pub y pem, y un cert autofirmado. TODO ESTO EN LA MAQUINA CA (ROCKY)

Tenemos que crear una autoridad (CA) propia con los siguientes parametros:
- **CountryName**: ES
- **StateOrProvinceName**: Madrid
- **LocalityName**: Alcorcon
- **OrganizationName**: CA de Pruebas

Empezamos creando la estructura necesaria:
```bash
cd /etc/pki/tls
sudo mkdir crl
sudo mkdir newcerts
sudo touch index.txt # # Base de datos de certificados emitidos
echo 01 > serial # Número de serie para el próximo certificado
echo 01 > crlnumber # Número de serie para la próxima CRL
```

Editamos el openssl.conf (/etc/pki/tls/openssl.cnf) para poner la ruta correcta y los valores predeterminados (no hace falta ponerlos, se pueden poner cuando se crea el certificado):
```cnf
# ...

[ CA_default ]

dir             = /etc/pki/tls          # ESTE DE AQUI!!!!!
certs           = $dir/certs            # Where the issued certs are kept
crl_dir         = $dir/crl              # Where the issued crl are kept
database        = $dir/index.txt        # database index file.
#unique_subject = no                    # Set to 'no' to allow creation of
                                        # several certs with same subject.

#### MAS PARA ABAJO ####

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = ES # Esto!!!
countryName_min                 = 2
countryName_max                 = 2

stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Madrid # Esto!!!

localityName                    = Locality Name (eg, city)
localityName_default            = Alcorcon # Esto!!!

0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = CA de Pruebas # Y esto!!!

```

Y ahora creamos una clave privada con passphrase (1234):
```bash
sudo openssl genrsa -aes256 -out private/cakey.pem 2048
# te pide que metas una passphrase, meter 1234
```

Y ahora creamos el certificado con los datos predeterminados (realmente no hace falta meterlos en el archivo se pueden poner ahora) y con CN = “CA del CEU para Pruebas” (cacert.pem):
```bash
sudo openssl req -new -key private/cakey.pem -out ca-csr.pem
```
Nos saldra esto, al haber puesto los valores predeterminados podemos solo darle a enter, si no se pueden poner ahora. **IMPORTANTE: Si hay que poner el Common Name (CN)**:
```txt
Enter pass phrase for private/cakey.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [ES]:
State or Province Name (full name) [Madrid]:
Locality Name (eg, city) [Alcorcon]:
Organization Name (eg, company) [CA de Pruebas]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:CA del CEU para pruebas # ESTO !!!
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
```bash
sudo openssl req -x509 -extensions v3_ca -in ca-csr.pem -out cacert.pem -key private/cakey.pem -days 3652
```

Creamos el listado de certificados revocados (crl.pem) que inicialmente estará vacío:
```bash
sudo openssl ca -gencrl -out crl.pem
```

La estructura por ahora es la siguiente:
```txt
[root@CARocky tls]# tree
.
├── cacert.pem
├── ca-csr.pem
├── cert.pem -> /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
├── certs
│   ├── ca-bundle.crt -> /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
│   ├── ca-bundle.trust.crt -> /etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt
│   └── localhost.crt
├── crl
├── crlnumber
├── crlnumber.old
├── crl.pem
├── ct_log_list.cnf
├── fips_local.cnf -> /etc/crypto-policies/back-ends/openssl_fips.config
├── index.txt
├── misc
├── newcerts
├── openssl.cnf
├── openssl.d
├── private
│   ├── cakey.pem
│   └── localhost.key
└── serial

6 directories, 16 files
```
---
---

### 4. Pareja de claves pub/priv RSA de 2048 bits para el server web y un certificado firmado por nuestra CA.

**CN y Nombre alternativo SAN**: www.miservidor.es

Editamos /etc/pki/tls/openssl.cnf para poner los nombres alternativos del SAN (lo metemos al final del archivo):
```conf
[ server_SAN ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = www.miservidor.es
DNS.2 = miservidor.es
```

Creamos la clave privada para el server de 2048 bits:
```bash
sudo openssl genrsa -out server-key.pem 2048
```

Y creamos el CSR y ponemos el CommonName = www.miservidor.es:
```bash
sudo openssl req -new -key server-key.pem -out server-csr.pem
```

Y ahora firmamos el CSR con nuestra CA:
```bash
sudo openssl ca -estensions server_SAN -in server-csr.pem -out server-crt.pem -days 730

# nos pide darle a 'y' para firmar
```

Lo siguiente es configurar el servidor Apache.

Creamos el directorio para el DocumentRoot, /projects/miservidor:
```bash
