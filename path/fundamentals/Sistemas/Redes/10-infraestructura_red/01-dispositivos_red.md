# Dispositivos de red

Los dispositivos de red son los componentes fisicos que interconectan, dirigen, filtran y gestionan el trafico de datos en una red. Cada dispositivo opera en una o varias capas del modelo OSI, y su eleccion determina la topologia, el rendimiento y la seguridad de la infraestructura.

Comprender las diferencias entre hubs, switches, routers y firewalls es fundamental para disenar y administrar cualquier red, desde una red domestica hasta un centro de datos empresarial.

## Hub (capa 1)

El hub (*repetidor*) es el dispositivo de red mas basico. Opera exclusivamente en la capa fisica: recibe una senal electrica por un puerto y la retransmite por todos los demas puertos sin interpretarla.

```
Puerto 1 → [HUB] → Puertos 2, 3, 4, 5, 6, 7, 8
```

**Caracteristicas:**

- Todos los puertos estan en el mismo dominio de colision.
- No aprende direcciones MAC; reenvia todo a todos.
- Full-duplex imposible (el medio es compartido).
- Prcticamente en desuso; los switches han reemplazado a los hubs por completo.

**Por que los hubs son obsoletos:** un hub de 24 puertos crea un unico dominio de colision de 24 dispositivos. Si dos dispositivos transmiten simultaneamente, se produce una colision y ambos deben reintentar, reduciendo el rendimiento efectivo. Un switch crea 24 dominios de colision independientes, eliminando las colisiones.

## Switch (capa 2)

El switch opera en la capa de enlace de datos. Aprende las direcciones MAC de los dispositivos conectados y reenvia las tramas solo al puerto de destino correspondiente, mejorando el rendimiento y la seguridad frente al hub.

```
MAC 00:1A → [SWITCH] → Puerto 3 (solo al destinatario)
```

**Caracteristicas:**

- Cada puerto es un dominio de colision independiente.
- Aprende direcciones MAC automaticamente (tabla CAM).
- Permite full-duplex (cada par de puertos puede transmitir y recibir simultaneamente).
- Soporta VLANs, STP, QoS en modelos administrables.
- Filtra trafico basandose en direcciones MAC.

## Router (capa 3)

El router opera en la capa de red. Utiliza direcciones IP para tomar decisiones de enrutamiento: que camino debe seguir un paquete para llegar a su destino a traves de multiples redes interconectadas.

```
Red 192.168.1.0/24 → [ROUTER] → Red 10.0.0.0/8
```

**Caracteristicas:**

- Dirige paquetes entre redes diferentes basandose en tablas de enrutamiento.
- Implementa protocolos de enrutamiento (OSPF, BGP, RIP).
- Proporciona NAT, DHCP relay y firewall basico.
- Segmenta dominios de broadcast.
- Toma decisiones paquete a paquete.

## Firewall

El firewall filtra el trafico de red segun reglas de seguridad configuradas por el administrador. Puede operar en multiples capas:

**Firewall de paquetes (capa 3-4):** filtra basandose en direcciones IP, puertos y protocolos. Rapido pero no puede inspeccionar el contenido.

```
# Regla: permitir HTTP y HTTPS, bloquear todo lo demas
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -j DROP
```

**Firewall de aplicacion (capa 7):** inspecciona el contenido de las peticiones HTTP/HTTPS. Puede bloquear inyeccion SQL, XSS y otros ataques de capa de aplicacion.

**Firewall de proximidad:** actua como intermediario entre el cliente y el servidor, inspeccionando y filtrando el trafico completo. Mas seguro pero introduce latencia adicional.

## Punto de acceso (AP) y modem-router combo

**Punto de acceso (AP):** dispositivo que crea una red inalambrica (WLAN) conectada a una red cableada existente. Los clientes Wi-Fi se conectan al AP, que reenvia el trafico a la LAN cableada.

**Modem-router combo:** dispositivo integrado que combina modem (modulador/demodulador para la conexion ISP) y router con AP Wi-Fi. Es el dispositivo tipico de los hogares y pequenas oficinas, proporcionando NAT, DHCP, firewall y Wi-Fi en un solo equipo.

> La eleccion del dispositivo adecuado depende del tamano de la red, los requisitos de rendimiento y seguridad, y el presupuesto. Una red domestica puede funcionar con un modem-router combo; una red empresarial necesita switches administrables, routers dedicados y firewalls de proximidad; y un centro de datos requiere switches de alta capacidad y routers de nucleo.

