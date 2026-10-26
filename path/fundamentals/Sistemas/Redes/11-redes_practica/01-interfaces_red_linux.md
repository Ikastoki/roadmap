# Interfaces de red en Linux

Linux expone las interfaces de red como archivos en el sistema de archivos y proporciona herramientas de linea de comandos para gestionarlas de forma completa. La comprension de las interfaces de red es el punto de partida para cualquier tarea de configuracion, diagnostico o administracion de red en Linux.

Los nombres de las interfaces en Linux siguen un esquema predecible basado en el tipo de hardware y la posicion en el bus, lo que facilita la identificacion del tipo de conexion (cableada, inalambrica, virtual).

## Nombres de interfaces

Los nombres de interfaces modernos en Linux siguen el esquema predictable introducido por systemd:

- **enp0s3:** Ethernet (en), bus PCI (p0), posicion 0, funcion 3. Típico en VMs VirtualBox.
- **wlp2s0:** Wi-Fi (wl), bus PCI (p2), posicion 0. Adaptador inalambrico.
- **enp0s8:** Segunda interfaz Ethernet en una VM.
- **lo:** interfaz de loopback (localhost, 127.0.0.1).
- **docker0, virbr0:** interfaces virtuales de Docker y KVM.

Los nombres antiguos (eth0, wlan0) pueden seguir apareciendo en sistemas antiguos o con configuracion manual.

## Comandos esenciales de red

### ip

`ip` es la herramienta moderna para gestionar interfaces, direcciones, rutas y vecinos. Reemplaza a las herramientas obsoletas `ifconfig`, `route` y `arp`.

```bash
# Ver interfaces y direcciones
ip addr show
ip -br addr show          # Formato breve

# Activar/desactivar interfaz
sudo ip link set eth0 up
sudo ip link set eth0 down

# Asignar direccion IP
sudo ip addr add 192.168.1.100/24 dev eth0

# Eliminar direccion IP
sudo ip addr del 192.168.1.100/24 dev eth0

# Ver tabla de enrutamiento
ip route show

# Añadir ruta estatica
sudo ip route add 10.0.2.0/24 via 10.0.1.1

# Ver vecinos ARP/NDP
ip neigh show
```

### ping

`ping` envia solicitudes ICMP Echo a un host y mide la latencia y la perdida de paquetes.

```bash
# Ping basico
ping google.com

# Numero limitado de pings
ping -c 4 8.8.8.8

# Intervalo personalizado
ping -i 0.2 192.168.1.1   # Un ping cada 200ms

# Tamaño del paquete
ping -s 1400 google.com    # Paquetes de 1400 bytes

# Flood ping (solo root, para pruebas de carga)
sudo ping -f 192.168.1.1
```

### traceroute

`traceroute` descubre los routers intermedios en el camino hacia un destino, enviando paquetes con TTL incremental.

```bash
traceroute google.com
traceroute -n 8.8.8.8      # Sin resolver nombres (mas rapido)
traceroute -T google.com    # Usa TCP en lugar de ICMP
```

### ss

`ss` (*Socket Statistics*) muestra conexiones de red activas, puertos en escucha y estadisticas de sockets.

```bash
# Puertos en escucha
ss -tuln

# Conexiones establecidas
ss -tan state established

# Con procesos asociados
ss -tulnp

# Estadisticas generales
ss -s
```

### nmcli

`nmcli` es la herramienta de linea de comandos de NetworkManager, el gestor de red predeterminado en la mayoria de distribuciones Linux modernas.

```bash
# Ver conexiones
nmcli connection show

# Activar una conexion
nmcli connection up "MiRed"

# Ver estado de Wi-Fi
nmcli device wifi list

# Conectarse a una red Wi-Fi
nmcli device wifi connect "MiRed" password "contrasena"

# Crear conexion con IP estatica
nmcli connection add type ethernet con-name "oficina" \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.method manual
```

> Las herramientas de red en Linux son extremadamente poderosas y flexibles. `ip` ha reemplazado a `ifconfig` y `route` como herramienta unificada de gestion. `nmcli` simplifica la configuracion en escritorios y portatiles. `ss` ha reemplazado a `netstat` con mejor rendimiento. Dominar estas herramientas es indispensable para cualquier administrador de sistemas Linux.

