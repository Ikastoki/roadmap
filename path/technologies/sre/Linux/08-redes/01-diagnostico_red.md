# Diagnostico de red en Linux

Un sistema Linux sin conectividad de red es, en la practica, una isla. La capacidad de diagnosticar problemas de red -- desde una interfaz mal configurada hasta un DNS que no responde -- distingue al administrador que reacciona con metodo del que actua por ensayo y error. Linux ofrece una pila de herramientas que cubren cada capa del modelo de red: desde la capa fisica (enlace) hasta la resolucion de nombres (aplicacion), y entender como encajan entre si es la clave para diagnosticar cualquier incidencia.

## Interfaces de red y direccionamiento IP

Toda comunicacion en red pasa por una interfaz. Las interfaces fisicas (`eth0`, `wlan0`) corresponden a hardware de red, mientras que `lo` (loopback) es una interfaz virtual que permite al sistema comunicarse consigo mismo a traves de la pila TCP/IP, util para pruebas locales y servicios que escuchan en `127.0.0.1`.

El comando `ip` es la herramienta moderna para gestionar interfaces y direcciones, reemplazando al obsoleto `ifconfig`:

```bash
ip addr show          # Muestra todas las interfaces y sus IPs
ip addr show eth0     # Muestra solo la interfaz eth0
ip -brief addr        # Vista resumida de todas las interfaces
ip link set eth0 up   # Activa una interfaz
ip link set eth0 down # Desactiva una interfaz
ip addr add 192.168.1.10/24 dev eth0  # Asigna una IP estatica
```

La salida de `ip addr` muestra informacion esencial: el estado de la interfaz (UP/DOWN), la direccion MAC, las direcciones IPv4 e IPv6 asignadas y el ambito (global, link). Una interfaz en estado DOWN no enviara ni recibira trafico, independientemente de que tenga una IP configurada.

La configuracion de red puede gestionarse mediante diferentes sistemas segun la distribucion:

- **Netplan** (Ubuntu 17.10+): archivos YAML en `/etc/netplan/`. Se aplican con `sudo netplan apply`.
- **ifupdown** (Debian clasico): archivo `/etc/network/interfaces`. Se aplica con `sudo ifup eth0`.
- **sysconfig** (RHEL/CentOS/Fedora): archivos en `/etc/sysconfig/network-scripts/ifcfg-eth0`.

> En sistemas modernos, NetworkManager abstrae la configuracion subyacente. La herramienta `nmcli` permite gestionar conexiones desde terminal sin necesidad de tocar archivos de configuracion manualmente.

## NetworkManager y nmcli

NetworkManager es el gestor de red por defecto en la mayoria de las distribuciones de escritorio y muchas de servidor. Su interfaz CLI, `nmcli`, permite ver, crear, modificar y eliminar conexiones de red:

```bash
nmcli dev status                  # Estado de los dispositivos de red
nmcli con show                    # Lista las conexiones configuradas
nmcli con show "MiRed"            # Detalles de una conexion
nmcli dev wifi list               # Escanea redes WiFi disponibles
nmcli dev wifi connect "SSID" password "clave"  # Conecta a WiFi
nmcli con up "MiRed"              # Activa una conexion
nmcli con down "MiRed"            # Desactiva una conexion
nmcli con add type ethernet ifname eth0  # Crea conexion ethernet
```

`nmcli` es especialmente util en entornos sin interfaz grafica o en scripts de automatizacion. Su salida es procesable y admite formato tabular, JSON o de clave-valor.

## Pruebas de conectividad: ping

`ping` envia paquetes ICMP Echo Request a un destino y espera respuestas ICMP Echo Reply. Es la primera herramienta para verificar conectividad basica:

```bash
ping 8.8.8.4             # Ping a una IP (Ctrl+C para detener)
ping -c 5 192.168.1.1    # Envia exactamente 5 paquetes
ping -4 google.com       # Fuerza IPv4
ping -6 google.com       # Fuerza IPv6
```

La interpretacion de la salida va mas alla de "responde o no responde". Perdidas de paquetes intermitentes indican congestion o problemas de enlace. Una latencia alta y variable (jitter) sugiere problemas de calidad en la ruta. La ausencia total de respuesta puede deberse a un cortafuegos que bloquea ICMP, no necesariamente a que el destino este caido.

> Algunos administradores configuran firewalls para no responder a ICMP, lo que hace que un ping fallido no sea diagnostico definitivo. En ese caso, `tcping` o `nc -zv` pueden probar puertos TCP concretos.

## Trazado de ruta: traceroute y mtr

Cuando la conectividad falla, saber donde se pierde el trafico es esencial. `traceroute` descubre la ruta que siguen los paquetes hasta un destino enviando paquetes con TTL (Time To Live) incremental:

```bash
traceroute google.com        # Trazado estandar
traceroute -n google.com     # Sin resolucion DNS (mas rapido)
traceroute -I google.com     # Usa ICMP en lugar de UDP
```

Cada salto representa un router intermedio. Si un salto muestra asteriscos (`* * *`) puede indicar que el router no responde a los paquetes de traceroute (configuracion habitual en muchos dispositivos de red), no necesariamente que la ruta este rota.

`mtr` combina `traceroute` con `ping` en una unica herramienta interactiva que actualiza continuamente las estadisticas de cada salto:

```bash
mtr google.com          # Vista interactiva en tiempo real
mtr -r -c 10 google.com # Modo reporte: 10 paquetes por salto
mtr -n google.com       # Sin resolucion DNS
```

La ventaja de `mtr` es que muestra perdida de paquetes por salto, permitiendo identificar rapidamente donde se produce la congestion o el descarte de trafico.

## Resolucion de nombres

La resolucion de nombres traduce nombres de dominio legibles a direcciones IP. Linux sigue un orden definido en `/etc/nsswitch.conf` (tipicamente: `hosts: files dns`), que determina si se consulta primero el archivo local o el servidor DNS.

El archivo `/etc/hosts` contiene asignaciones estaticas de nombre a IP. Es util para sobrescribir resoluciones DNS o para entornos sin servidor DNS:

```bash
cat /etc/hosts
127.0.0.1   localhost
192.168.1.5 servidor-backup
```

En sistemas modernos con `systemd-resolved`, las consultas DNS se gestionan a traves de un servicio local que cachea respuestas y soporta DNSSEC. `resolvectl` es la herramienta para interactuar con el:

```bash
resolvectl status             # Estado del resolvedor
resolvectl query google.com   # Resuelve un nombre
resolvectl dns eth0           # Muestra los DNS de una interfaz
```

Las herramientas clasicas `dig` y `nslookup` realizan consultas DNS directas, ignorando el cache local:

```bash
dig google.com                # Consulta completa con todas las secciones
dig +short google.com         # Solo la direccion IP
dig -x 8.8.8.8                # Resolucion inversa (PTR)
nslookup google.com           # Consulta simple
nslookup google.com 8.8.8.8   # Consulta usando un DNS especifico
```

`dig` es preferible para diagnostico porque muestra informacion detallada: el servidor que respondio, el tiempo de respuesta, los flags, las secciones de autoridad y los registros adicionales.

