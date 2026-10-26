# Funcion de la capa de red

La capa de red es responsable del enrutamiento y el direccionamiento logico de los paquetes entre redes diferentes. Mientras que la capa de enlace se encarga de la comunicacion entre dispositivos en la misma red local (usando direcciones MAC), la capa de red permite que los datos viajen a traves de multiples redes interconectadas hasta llegar a su destino final, donde sea que este.

La capa de red es la que hace posible internet: sin ella, cada red local estaria aislada y no seria posible la comunicacion entre dispositivos en diferentes partes del mundo. Los routers operan en esta capa, tomando decisiones de enrutamiento basadas en direcciones IP.

## Enrutamiento

El enrutamiento es el proceso de determinar el camino optimo que deben seguir los paquetes desde su origen hasta su destino. Los routers utilizan tablas de enrutamiento que contienen informacion sobre las redes alcanzables y el mejor camino hacia ellas.

Los algoritmos de enrutamiento calculan las mejores rutas considerando metricas como el numero de saltos (*hop count*), el ancho de banda disponible, la latencia, la carga actual y la fiabilidad de los enlaces. Los protocolos de enrutamiento (OSPF, BGP, RIP) actualizan las tablas de enrutamiento cuando la topologia de la red cambia.

## Direccionamiento logico

Las direcciones IP son la base del direccionamiento logico en capa 3. A diferencia de las direcciones MAC (fijas por el fabricante), las direcciones IP son configurables y pueden cambiar dependiendo de la red a la que se conecte el dispositivo.

Cada interfaz de red conectada a una red IP tiene asignada una direccion IP unica dentro de esa red. Las direcciones IP permiten a los routers identificar la red de origen y la red de destino de cada paquete, y tomar decisiones de enrutamiento en consecuencia.

## Fragmentacion

Cuando un paquete es demasiado grande para ser transmitido a traves de un enlace con una MTU (*Maximum Transmission Unit*) menor, el router o el host origen debe dividirlo en fragmentos mas pequenos. Cada fragmento se transmite individualmente y se reensambla en el destino.

IPv4 permite la fragmentacion en routers intermedios, lo que puede causar problemas de rendimiento y fiabilidad. IPv6 elimino la fragmentacion en routers: solo el host origen puede fragmentar paquetes, lo que simplifica el procesamiento en los routers y mejora el rendimiento.

## Redes publicas y privadas

Las direcciones IP se clasifican en publicas y privadas:

**Direcciones IP publicas:** unicas en toda internet, asignadas por IANA a los proveedores de servicios de internet (ISPs). Son enrutables en la red global y permiten la comunicacion directa entre dispositivos en diferentes partes del mundo.

**Direcciones IP privadas:** reservadas para uso dentro de redes locales y no son enrutables en internet. Los rangos definidos por RFC 1918 son:

- 10.0.0.0/8 (10.0.0.0 a 10.255.255.255) - Clase A
- 172.16.0.0/12 (172.16.0.0 a 172.31.255.255) - Clase B
- 192.168.0.0/16 (192.168.0.0 a 192.168.255.255) - Clase C

La comunicacion con internet se realiza mediante NAT (*Network Address Translation*), que traduce direcciones privadas a publicas y viceversa.

> La capa de red es donde se toman las decisiones mas criticas de una comunicacion: que camino debe seguir un paquete para llegar a su destino. Estas decisiones determinan la latencia, la fiabilidad y el rendimiento de la comunicacion. Un enrutamiento inadecuado puede causar paquetes perdidos, latencia elevada y experiencias de usuario pobres, incluso cuando la infraestructura fisica es excelente.

