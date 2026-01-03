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
