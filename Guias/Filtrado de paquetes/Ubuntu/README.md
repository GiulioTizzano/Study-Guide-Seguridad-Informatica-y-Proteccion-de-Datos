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

**2. Se nos pide comprobar el efecto que tiene el orden de las reglas de filtrado. Partiendo de una configuración sin filtros, hay que bloquear todo el tráfico TCP saliente salvo el dirigido a una dirección IP concreta de nuestra subred, viendo el efecto del orden en la entrada de las dos reglas necesarias.**

Configuración sin filtros:
```

# Reestablecer reglas
sudo iptables -F
sudo iptables -X

# Reestablecer políticas
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
```

Bloquear todo el tráfico TCP saliente salvo el dirigido a una dirección IP concreta de nuestra subred, viendo el efecto del orden en la entrada de las dos reglas necesarias:
```
# Este es el caso correcto porque iptables evalúa las reglas en orden secuencial y aplica la primera coincidencia, sin evaluar el resto. Por tanto, si aceptamos una única conexión y luego bloqueamos el resto entonces funcionará.
iptables -A OUTPUT -p tcp -d 192.168.126.149 -j ACCEPT
iptables -A OUTPUT -p tcp -j DROP
```

```
# Caso incorrecto (general antes que específica)
sudo iptables -A OUTPUT -p tcp -j DROP
sudo iptables -A OUTPUT -p tcp -d 192.168.126.149 -j ACCEPT

# Este no funciona porque si bloquea todo, luego deja de evaluar el resto de reglas porque siempre evalúa la primera regla genérica!

NOTA: Nunca poner una regla genérica sin condiciones antes que las reglas específicas.
```

**3. Definir una regla que redirija todo el tráfico SSH saliente que va hacia una dirección IP concreta (por ejemplo, 1.2.3.4) para que se redirija a la IP de un servidor real que tenga el servicio SSH activo. Para probar su funcionamiento, abrir una sesión SSH contra 1.2.3.4.**

Básicamente tenemos que hacer un port-forwarding para que cuando intentemos acceder a la ip 1.2.3.4 realmente estemos haciendo ssh a la 192.168.126.149 (en mi caso de subred específica):
```
iptables -t nat -A OUTPUT -p tcp -d 1.2.3.4 --dport 22 -j DNAT --to-destination 192.168.126.149:22

```

Prueba:
```
ssh root@1.2.3.4 
```
Deberíamos poder conectarnos

**4. Desde una máquina remota, utilizar **nmap** para identificar el SO de nuestra máquina virtual Linux. Luegom aplicar una regla **CON ESTADO** para impedir conexiones TCP inválidas (las que no se inician con el segmento SYN). Volver a utilizar la herramienta nmap para identificar el SO y comentar los nuevos resultados obtenidos**

Escaneo del SO con nmap antes de aplicar la regla con estado:
```
sudo nmap -O 192.168.126.149
```

Ahora aplicamos la regla CON ESTADO para impedir conexiones TCP inválidas (las que no se inician con el segmento SYN). Luego, volver a utilizar la herramienta nmap para identificar el SO y comentar los nuevos resultados obtenidos:
```
# Regla con estado para impedir conexiones TCP inválidas
iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
```
Ahora no es posible ver la información acerca del SO.

**5. Ahora hay que partir de nuevo de una configuración sin filtros y bloquear todo el tráfico entrante y saliente. Luego, definir reglas CON ESTADO para permitir iniciar conexiones TCP o UDP hacia el exterior, así como el tráfico ICMP siempre que sea iniciado por nosotros o relacionado con nuestras conexiones. Hacer comprobaciones iniciando conexiones exteriores, intentando conectar desde el exterior (por ejemplo al servidor SSH o recibiendo mensajes ping).**

Configuración sin filtro y bloqueando todo tráfico entrante y saliente:
```
sudo iptables -F
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
sudo iptables -P FORWARD DROP
```

Reglas con estado para permitir iniciar conexiones TCP o UDP hacia el exterior, así como tráfico ICMP iniciado por nosotros o relacionado con nuestras conexiones:
```
# Regla con estado TCP:
sudo iptables -A OUTPUT -p tcp -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

sudo iptables -A INPUT -p tcp -m state --state ESTABLISHED, RELATED -j ACCEPT

# Regla con estado UDP

iptables -A OUTPUT -p udp -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -A INPUT  -p udp -m state --state ESTABLISHED,RELATED -j ACCEPT

# Regla con estado icmp

iptables -A OUTPUT -p icmp -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -A INPUT  -p icmp -m state --state ESTABLISHED,RELATED -j ACCEPT

```

Cuando hacemos ping desde la máquina con las máquinas con estado si que permite hacer ping y recibir respuesta. Pero desde el exterior no permite hacer ping.

**6. **






