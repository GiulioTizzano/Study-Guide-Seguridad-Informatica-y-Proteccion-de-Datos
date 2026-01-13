# Práctica 4 Filtrado de Paquetes:

## Cortafuegos personal
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

**6. Considere la siguiente situación: Se están produciendo accesos continuos a nuestro
servidor OpenSSH para intentar descubrir mediante fuerza bruta usuarios y contraseñas
válidas del sistema. Nos gustaría limitar el número de accesos desde una única dirección
IP origen (por ejemplo, un máximo de 3 conexiones cada 120 segundos desde cada IP
origen), para minimizar el riesgo este tipo de intentos sin afectar a los usuarios legítimos.
Busque una solución para este problema mediante iptables, e indique la(s) regla(s) que
habría que aplicar para implantar esta política y la función que cumplen. Para acotar más
aún estos intentos de acceso es posible limitar también el número máximo de reintentos de autentificación en cada sesión (por ejemplo, a dos). ¿Cómo podríamos configurar esta nueva restricción? **

```
# 1. Registrar cada intento SSH nuevo por IP origen
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set --name LISTASSH --rsource

2. Bloquear si una IP supera 3 intentos en 120 segundos
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 120 --hitcount 4 --name LISTASSH --rsource -j DROP

# 3. Permitir el acceso SSH si no se supera el límite
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT
```

Explicación funcional de las reglas
Regla	Función
Regla 1	Registra la IP origen de cada intento SSH nuevo
Regla 2	Comprueba si la IP ha superado 3 intentos en 120 s y bloquea
Regla 3	Permite el acceso SSH a IPs que no superan el límite

Como iptables no tiene capacidad para acotar el número máximo de intentos de autentificación en cada sesión, podemos entonces usar el archivo de configuración de ssh que se encuentra en el directorio **/etc/ssh/sshd_config**:
```
# Y editamos el campo:
MaxAuthTries 2
```

Reiniciar el servicio ssh:
```
sudo systemctl restart ssh
```

## Cortafuegos personal con UFW

**7. UFW (Uncomplicated Firewall) es una aplicación desarrollada por Canonical (Ubuntu) para
activar reglas iptables y mantener un cortafuegos personal de manera sencilla. Para
realizar este apartado se recomienda usar una máquina Ubuntu, que ya tienen instalado
UFW. Llevar a cabo las siguientes tareas:**

**a. Verificar el estado del cortafuegos personal y si ya existen reglas iptables**
```
# Comprobar estado
sudo ufw status
```

```
# Comprobamos si hay reglas por defecto
sudo iptables -L -n
```

**b. Arrancar algún servicio en el equipo para realizar las pruebas posteriores (Apache2,OpenSSH Server, …) **
```
# Arrancamos el servidor apache y ssh para realizar pruebas para luego:
sudo systemctl status apache2
sudo systemctl status ssh
```

**c. Arrancar UWF y configurar su arranque automático. Iniciar el cortafuegos (ufw enable)**

```
sudo systemctl enable ufw
sudo ufw enable
```

**d. Comprobar el estado de UFW (ufw status). También es posible ver información
extendida (ufw status verbose) que incluye la política definida, y las reglas iptables
generadas (iptables -L -n | more). Con este último comando podemos ver las nuevas reglas y cadenas generadas por UFW.**
```
# Comprobar el estado de UFW
sudo ufw status

# Ver información extendida
sudo ufw verbose

# Reglas iptables generadas
sudo iptables -L -n | more
```
UFW es capaz de traducir automáticamente sus reglas a reglas de iptables.

**e. Usar el comando ufw default allow|deny incoming|outgoing para definir la configuración
por defecto del cortafuegos personal. Comprobar el efecto de estos comandos intentando conexiones hacia o desde el equipo. **

```
# Restringimos conexiones salientes y entrantes
sudo ufw default deny incoming

sudo ufw defauly deny outgoing
```

```
# Volvemos a permitir la salida y entrada de tráfico
sudo ufw default allow incoming

sudo ufw default allow outgoing
```
Se puede escoger la combinación que queramos con UFW

**f. Establecer una política por defecto que permita conexiones salientes e impida conexiones entrantes.**
```
# Denegar conexiones entrantes
sudo ufw default deny incoming

# Aceptar conexiones salientes
sudo ufw defauly allow outgoing
```
```
# Comprobar configuración
sudo ufw status verbose
```

**g. Sobre la configuración anterior permitir conexiones entrantes a los servicios arrancados en el apartado b) (ufw allow …)**

```
# Permitir ssh
sudo ufw allow ssh

# Permitir http
sudo ufw allow http

# Ver estado
sudo ufw status 
```

**h. También es posible limitar las conexiones realizadas desde una IP origen (ufw limit …), para mitigar los ataques de diccionario. Activar el acceso limitado al servidor SSH de nuestro equipo. ¿Qué límite se ha establecido para las conexiones SSH entrantes?**
```
# Limitar tráfico ssh
sudo ufw limit ssh

# limitar http
sudo ufw limit http

# Comprobar estado
sudo ufw status
```
De forma predeterminada se establece un límite de 6 conexiones cada 30 segundos a nuestra máquina.

## Router con función de cortafuegos y NAT
**8. 8. Vamos a preparar una maqueta como la reflejada en el gráfico adjunto. Usaremos tres máquinas virtuales (FW, interna y externa) y asociaremos un segundo interface a la
máquina que actúa como cortafuegos. Se recomienda usar el SO Ubuntu Server al menos en FW, ya que cuenta con los paquetes shorewall en el repositorio oficial. La red exterior tendrá un direccionamiento público (20.20.20.0/24) y la privada tendrá direccionamiento privado (10.10.10.0/24) (puede usar las redes predefinidas del entono de virtualización para ambas redes). Las máquinas interna y externa tendrán como router por defecto al
equipo que actúa como firewall (FW). **

![Diagrama/maqueta dispositivos](/Guias/Filtrado%20de%20paquetes/images/diagrama.png)

Para este apartado habría que retocar el direccionamiento entrando en el netplan de las máquinas ubuntu pero no lo haremos porque en el examen no tiene sentido cambiar el direccionamiento porque sino perderemos conectividad con las máquinas.

**9. Instalar shorewall en el equipo FW. También se recomienda instalar la herramienta tcpdump y algún servicio (por ejemplo, Apache) en las tres máquinas. 
**

```
# Dentro de la máquina firewall instalamos shorewall, apache y tcpdump

sudo apt install shorewall -y
sudo apt install apache2 -y
sudo apt install tcpdump -y
```

```
# Dentro del resto de máquinas instalamos apache y tcpdump
sudo apt install apache2 -y
sudo apt install tcpdump -y
```

**10. Configurar los interfaces de red de los tres equipos y default GW en los hosts interno y
externo. Reiniciar los servicios de red y comprobar que somos capaces de alcanzar las máquinas interna y externa desde el cortafuegos (FW). Habilitar el encaminamiento en el cortafuegos (net.ipv4.ip_forward=1 en el archivo /etc/sysctl.conf) y comprobar que es posible comunicar las máquinas interna y externa sin restricciones. Levantar algún servicio en los equipos y comprobar que es posible acceder remotamente. **

```
# Habilitar el encaminamiento en el cortafuegos entrando al archivo /etc/sysctl.conf poniendo el valor net.ipv4.ip_forward=1.

sudo vi /etc/sysctl.conf
```

```
# Comprobar que las máquinas internas y externa se puedan comunicar sin restricciones

# Interna --> externa
ping 20.20.20.150

# Externa --> interna
ping 10.10.10.150
```

Levantar algún servicio en los equipos y comprobar que es posible acceder remotamente:
```
# Tanto la máquina interna como la externa tienen un servidor apache corriendo con la configuración predeterminada. Por tanto, ahora haremos un curl/lynx para ver si es accesible el contenido sea accesible para la interna hacia la externa y vice versa:

# Interna --> externa
curl 20.20.20.150
# Externa --> interna
curl 10.10.10.150
```

** 11. Describir unas políticas básicas en el cortafuegos Shorewall: se acepta el tráfico desde la red interna al exterior (saliente), se impide tráfico entrante desde el exterior, se acepta tráfico saliente desde el cortafuegos hacia cualquier sitio. Se acepta tráfico entrante al
cortafuegos desde la red interna. Activar las reglas (systemctl start shorewall) y verificar su
correcto funcionamiento. 
**

Entramos dentro de la máquina FW y iniciamos y activamos el demonio de shorewall en caso de que no esté corriendo:
```
sudo systemctl status shorewall
sudo systemctl enable shorewall
sudo systemctl start shorewall
```
Luego, vemos los archivos que hay bajo el directorio /etc/shorewall/:
```
sudo ls /etc/shorewall
```

```
Deberían poderse ver los ficheros siguientes:

conntrack interfaces params policy rules shorewall.conf zones

En caso de que no estén, podemos ir al directorio:

cd /usr/share/doc/shorewall/examples

Ahí podremos mover a nuestro directorio /etc/shorewall algunos ejemplos más comúnes de configuración de los ficheros que necesitamos en caso de que no estén ahí.
```

Ahora explicaremos brevemenete para que sirve cada fichero para entenderlo:
```
Archivo zones --> las zonas en las que se descompone la red que ve shorewall:
loc: zona privada/local
dmz: zona desmilitarizada donde se encuentran los recursos restringidos de tu red pero con acceso a Internet
net: zona insegurda, habitualmente Internet
fw: nuestro propio equipo (que actua de firewall)

Archivo policy --> archivo donde se definen las políticas por defecto

Archivo rules --> define excepciones a las políticas por defecto

Archivo interfaces --> lista de interfaces de red del equipo, asociadas a las zonas en las que están presentes

Archivo masq/snat --> establece el enmascaramiento de las direcciones IP (dirección privada --> pública)

Archivo dnat --> para el port-forwarding
```

Se nos pide lo siguiente **Aceptar el tráfico desde la red interna al exterior (saliente) , se impide tráfico entrante desde el exterior, se acepta tráfico saliente desde el cortafuegos hacia cualquier sitio. Se acepta tráfico entrante al cortafuegos desde la red interna.**:
```
Para ello, primero preparamos los ficheros en /etc/shorewall:

sudo mv /usr/share/doc/shorewall/examples/two-interfaces/zones /etc/shorewall
sudo mv /usr/share/doc/shorewall/examples/two-interfaces/interfaces /etc/shorewall
sudo mv /usr/share/doc/shorewall/examples/two-interfaces/policy /etc/shorewall

```

Luego, en el archivo **policy** ponemos una regla para **aceptar el tráfico desde la red interna al exterior**:
```
###############################################################################
#SOURCE DEST            POLICY          LOGLEVEL        RATE    CONNLIMIT

loc     net             ACCEPT
net     loc             DROP
fw      all             ACCEPT          $LOG_LEVEL
loc     fw              ACCEPT
# THE FOLOWING POLICY MUST BE LAST
all     all             REJECT          $LOG_LEVEL
```
Luego, verificamos que se hayan aplicado correctamente las reglas haciendo:
```
Desde la interna -> externa:
ping 20.20.20.150

Debería funcionar

Desde la externa -> interna:
ping 10.10.10.150

No debería funcionar los pings
```

**12. Habilitar el enmascaramiento de direcciones desde la red interior hacia la red exterior.
Verificar que todo el tráfico saliente se enmascara (puede usar el sniffer tcpdump). Se recomienda desactivar el GW en la máquina Externa y comprobar que sigue siendo posible alcanzar sus servicios desde el interior**.

Para habilitar el enmascaramiento tenemos que editar el fichero /etc/shorewall/snat y meterle la subred de la dirección IP pública con su correspondiente dirección IP:
```
MASQUERADE          10.10.10.0/24       NET_IF
```

Quitar el gateway por defecto para la máquina externa para comprobar su funcionamiento correcto. Luego, desde la máquina interna comprobamos que llegen los ping, y que desde la externa a la interna no:
```
Interna -> externa:
ping 20.20.20.150

Debería funcionar

Externa --> interna:
ping 10.10.10.150

No debería funcionar
```

**13. Implantar una política restrictiva para el tráfico del FW: todo el tráfico entrante, saliente y que atraviesa el FW será rechazado. A continuación, se  habilitará la salida a servicios concretos (http, https, SMTP, …). Configurar algún servicio de port forwarding (como Apache) que será visible desde el exterior y ofrecido por la máquina de la red interna. Configurar un acceso al cortafuegos por SSH desde una dirección externa concreta. Establecer el redireccionamiento de todo el tráfico DNS saliente para que se reenvíe a un servidor de DNS externo concreto.
Para realizar este apartado se recomienda estudiar los ejemplos de reglas shorewall presentes en el manual de esta aplicación (man shorewall-rules). **




