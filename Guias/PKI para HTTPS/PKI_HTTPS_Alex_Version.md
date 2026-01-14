
# Guía Práctica: PKI para HTTPS (Apache, OpenSSL y EJBCA)

Esta guía detalla la implementación de una infraestructura de clave pública (PKI) para asegurar servicios web mediante HTTPS, incluyendo la creación de una CA propia y la gestión de certificados.

---

## 1. Preparación del Sistema

### Configuración de Fecha y Hora
Es crítico que el servidor tenga la hora correcta para que los certificados no nazcan "caducados".
```bash
# Sincronizar con servidor NTP
sudo ntpdate -u pool.ntp.org
# O manualmente
sudo date -s "2024-05-20 10:00:00"
```

### Instalación de Apache y Módulo SSL

```Bash
sudo apt update
sudo apt install -y apache2 openssl

# Activar módulo SSL y el sitio por defecto
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo systemctl restart apache2
```

---

## 2. Creación de una CA (Autoridad de Certificación) propia

Usaremos OpenSSL para actuar como nuestra propia entidad certificadora.

### Preparación de la estructura

```Bash
mkdir ~/mi_pki && cd ~/mi_pki
mkdir certs crl newcerts private
touch index.txt
echo 1000 > serial
```

### Generación de la CA (Clave y Certificado Raíz)

```Bash
# Generar clave privada de la CA (con frase de paso)
openssl genrsa -aes256 -out private/cakey.pem 4096

# Crear certificado autofirmado de la CA
# Datos sugeridos: C=ES, ST=Madrid, L=Alcorcon, O=CA de Pruebas, CN=CA del CEU para Pruebas
openssl req -new -x509 -days 3650 -key private/cakey.pem -out cacert.pem
```

### Creación del Listado de Revocación (CRL)

```Bash
openssl ca -gencrl -out crl/crl.pem
```

---

## 3. Certificado para el Servidor Web

### Generar Clave y Petición de Firma (CSR)

Para el servidor `www.miservidor.es`:

```Bash
# 1. Generar clave privada del servidor (sin contraseña para que arranque solo)
openssl genrsa -out private/server.key 2048

# 2. Generar el CSR (Petición de certificado)
# IMPORTANTE: En Common Name (CN) poner: www.miservidor.es
openssl req -new -key private/server.key -out server.csr
```

### Firmar el certificado con nuestra CA

```Bash
# La CA firma la petición del servidor
openssl ca -in server.csr -out certs/server.crt -days 365
```

---

## 4. Configuración de Apache para HTTPS

Editar el archivo de sitio SSL: `/etc/apache2/sites-enabled/default-ssl.conf`

```Python
<VirtualHost *:443>
    ServerName www.miservidor.es
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile    /home/usuario/mi_pki/certs/server.crt
    SSLCertificateKeyFile /home/usuario/mi_pki/private/server.key
    SSLCertificateChainFile /home/usuario/mi_pki/cacert.pem
</VirtualHost>
```

### Redirección HTTP a HTTPS

Editar `/etc/apache2/sites-available/000-default.conf`:

```Python
<VirtualHost *:80>
    ServerName www.miservidor.es
    Redirect permanent / [https://www.miservidor.es/](https://www.miservidor.es/)
</VirtualHost>
```

---

## 5. Confianza en el Cliente (Navegador)

Para evitar el error "La conexión no es privada":

1. Exportar el archivo `cacert.pem` al equipo cliente.
2. **Windows/Chrome:** Importar en "Entidades de certificación raíz de confianza".
3. **Firefox:** Ajustes -> Privacidad y Seguridad -> Certificados -> Ver certificados -> Importar.
---

## 6. Gestión de Revocación (CRL)

Si el certificado del servidor se ve comprometido:

```Bash
# 1. Revocar el certificado
openssl ca -revoke certs/server.crt

# 2. Generar CRL actualizada
openssl ca -gencrl -out crl/crl.pem

# 3. Configurar Apache para que verifique la CRL
# Añadir en la config de Apache:
# SSLCARevocationFile /home/usuario/mi_pki/crl/crl.pem
```

---

## 7. EJBCA - Instalación Avanzada (Resumen)

EJBCA es una solución empresarial de PKI basada en Java.

### Requisitos previos

```Bash
# Instalación de Java y MariaDB
sudo apt install default-jdk mariadb-server ant ant-optional

# Configuración de base de datos
sudo mysql -u root -p
> CREATE DATABASE ejbca CHARACTER SET utf8 COLLATE utf8_bin;
> GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost' IDENTIFIED BY 'ejbca_password';
```

### Despliegue con Wildfly

```Bash
# Descargar e instalar Wildfly
cd /opt
sudo wget [https://github.com/wildfly/wildfly/releases/download/26.1.3.Final/wildfly-26.1.3.Final.tar.gz](https://github.com/wildfly/wildfly/releases/download/26.1.3.Final/wildfly-26.1.3.Final.tar.gz)
sudo tar -xzf wildfly-26.1.3.Final.tar.gz

# Configurar EJBCA
cd /opt/ejbca/ejbca-ce
# Editar archivos de propiedades en /conf (p. ej. ejbca.properties)
ant deploy
ant install
```

---

## 8. Formatos de Certificado comunes

|**Formato**|**Extensión**|**Uso**|
|---|---|---|
|**PEM**|`.pem`, `.crt`, `.key`|Base64. Estándar en Linux/Apache.|
|**PKCS#12**|`.p12`, `.pfx`|Binario. Contiene certificado + clave. Muy usado en Windows/Navegadores.|
|**DER**|`.der`|Binario. Usado habitualmente en Java o dispositivos móviles.|

**Convertir de PEM a PKCS#12:**

Bash

```
openssl pkcs12 -export -out certificado.p12 -inkey server.key -in server.crt -certfile cacert.pem
```