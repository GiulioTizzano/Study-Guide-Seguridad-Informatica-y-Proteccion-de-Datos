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




