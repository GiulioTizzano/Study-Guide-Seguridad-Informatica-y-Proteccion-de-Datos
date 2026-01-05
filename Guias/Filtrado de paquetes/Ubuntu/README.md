# Práctica 4 Filtrado de Paquetes:

**1. Se nos pide establecer distinas reglas de filtrado **sin estado** sobre las cadenas INPUT y OUTPUT . Filtrar ambos sentidos de la conexión, en primer lugar realizar la política DROP para el tráfico entrante y saliente del equipo. A continuación, habilitar el intercambio ICMP con el resto de equipos de nuestra subred. Permitir el tráfico saliente hacia los puertos 80/tcp, 443/tcp y 53/udp (nuestro equipo actua como cliente). Luego, habilitar el tráfico entrante al puerto 22/tcp y 80/tcp (donde nuestro equipo actúa como servidor).**

```
# Comandos para borrar las reglas previas
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -X
```
```
# Comprobaciones:
sudo iptables -L -n
```

Política DROP para el tráfico entrante y saliente del equipo:
```
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

```

Además, habilitar el intercambio ICMP con el resto de equipos de nuestra subred:
```
iptables -A INPUT  -p icmp -s 192.168.126.0/24 -j ACCEPT
iptables -A OUTPUT -p icmp -d 192.168.126.0/24 -j ACCEPT
```

Checkear las reglas aplicadas:
```
sudo iptables -L -n
```

Permitir el tráfico saliente hacia los puertos 80/tcp, 443/tcp y 53/udp (nuestro equipo actúa como cliente). Habilitar el tráfico entrante al puerto 22/tcp y 80/tcp (nuestro equipo actúa como servidor).

```
# Actuando como cliente, habilitando el tráfico hacia los puertos:
# Puerto 80/tcp
sudo iptables -A OUTPUT -p tcp --dport 80  -j ACCEPT

# Puerto 443/TCP
sudo iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT

# Puerto 53/udp:
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT

# Puerto 80/tcp
sudo iptables -A INPUT -p tcp --sport 80  -j ACCEPT

# Puerto 443/TCP
sudo iptables -A INPUT -p tcp --sport 443 -j ACCEPT

# Puerto 53/udp:
sudo iptables -A INPUT -p udp --sport 53 -j ACCEPT

```


```
# Actuando como servidor, tráfico entrante hacia los puertos:

# Puerto 22/tcp:
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

# Puerto 80/tcp:
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT

```

Importante tener en cuenta que cuando se habla de tráfico SALIENTE lo estamos hablando desde el punto de vista de un CLIENTE. Cuando hablamos de tráfico ENTRANTE, entonces es desde el punto de vista del SERVIDOR.





