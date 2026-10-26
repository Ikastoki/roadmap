# Switches y su funcionamiento

Los switches son dispositivos de capa 2 que interconectan dispositivos en una red local Ethernet. A diferencia de los hubs (que reenvian todos los paquetes a todos los puertos), los switches aprenden que dispositivo esta conectado a cada puerto y reenvian las tramas solo al puerto de destino, mejorando significativamente el rendimiento y la seguridad de la red.

Los switches son el elemento fundamental de las LAN modernas: casi todas las redes cableadas utilizan switches para interconectar dispositivos, creando topologias en estrella que permiten comunicacion full-duplex sin colisiones.

## Tabla MAC (CAM table)

El switch mantiene una **tabla MAC** (tambien llamada tabla CAM - *Content Addressable Memory*) que asocia direcciones MAC con puertos. Esta tabla se construye dinamicamente mediante un proceso de aprendizaje:

1. Cuando el switch recibe una trama en un puerto, registra la direccion MAC origen y el puerto en su tabla MAC.
2. Cuando necesita reenviar una trama, busca la direccion MAC destino en la tabla.
3. Si la encuentra, reenvia la trama solo al puerto correspondiente.
4. Si no la encuentra (*unknown unicast*), reenvia la trama a todos los puertos excepto al de origen (*flooding*).

```
Tabla MAC del switch:
+------------------+--------+
| MAC Address      | Puerto |
+------------------+--------+
| 00:1A:2B:3C:4D:5E|   1    |
| 00:1A:2B:3C:4D:5F|   3    |
| 00:1A:2B:3C:4D:60|   7    |
+------------------+--------+
```

Las entradas de la tabla MAC envejecen tras un tiempo de inactividad (tipicamente 300 segundos) para evitar que la tabla crezca indefinidamente con dispositivos que ya no estan conectados.

## Flooding, forwarding y filtering

**Flooding (*inundacion*):** reenvio de una trama a todos los puertos excepto al de origen. Se produce cuando:
- La trama es broadcast (MAC destino FF:FF:FF:FF:FF:FF).
- La trama es multicast y el switch no tiene informacion del grupo.
- La direccion MAC destino no esta en la tabla MAC (*unknown unicast*).

**Forwarding (*reenvio*):** reenvio de una trama solo al puerto correcto, segun la tabla MAC. Es el comportamiento deseado y el mas eficiente, ya que la trama solo consume ancho de banda en el puerto de origen y el de destino.

**Filtering (*filtrado*):** retencion de una trama en el switch sin reenviarla. Ocurre cuando la direccion MAC destino y origen estan en el mismo puerto (la trama ya llego a su destino por otro camino). Esto reduce el trafico innecesario.

## Dominio de colision vs dominio de broadcast

**Dominio de colision:** conjunto de dispositivos donde puede producirse una colision si dos dispositivos transmiten simultaneamente. En un hub, todos los puertos estan en el mismo dominio de colision. En un switch, cada puerto es un dominio de colision independiente, eliminando las colisiones en Ethernet moderno.

**Dominio de broadcast:** conjunto de dispositivos que reciben las tramas broadcast de todos los demas. Todos los puertos de un switch estan en el mismo dominio de broadcast por defecto. Para segmentar dominios de broadcast, se utilizan VLANs.

Un switch de 24 puertos tiene 24 dominios de colision (uno por puerto) pero un unico dominio de broadcast (todos los puertos). Una VLAN divide el switch en multiples dominios de broadcast logicicos.

## STP (Spanning Tree Protocol)

STP (IEEE 802.1D) previene bucles en topologias con redundancia. Cuando multiples switches estan interconectados con enlaces redundantes, se pueden crear bucles que provocan *broadcast storms* (tormentas de broadcast) que saturan la red.

STP construye un arbol sin bucles (*spanning tree*) bloqueando logicamente los enlaces redundantes. Si un enlace activo falla, STP recalcula el arbol y activa un enlace bloqueado para mantener la conectividad.

```
Antes de STP:          Despues de STP:
A --- B                A --- B
|   / |                |     |
|  /  |                |     |
C --- D                C --- D
 (bucle)               (enlace C-D bloqueado)
```

Variantes modernas: RSTP (802.1w, convergencia rapida) y MSTP (802.1s, multiples instancias STP).

## VLANs

Las VLANs (*Virtual LANs*) permiten segmentar logicamente un switch fisico en multiples redes independientes. Los dispositivos en diferentes VLANs no pueden comunicarse directamente sin un router, incluso si estan conectados al mismo switch fisico.

Las VLANs mejoran la seguridad (aislamiento de trafico), reducen el dominio de broadcast y facilitan la organizacion logica de la red. Se configuran en el switch asignando puertos a IDs de VLAN (1-4094) y utilizando etiquetado 802.1Q para trafico entre switches (*trunking*).

> Los switches son el corazon de las LAN modernas. Un switch administrable de gama alta puede gestionar miles de dispositivos con QoS, spanning tree, ACLs y monitoreo de trafico. La eleccion del switch adecuado depende del numero de puertos, la velocidad, las funcionalidades de gestion y el presupuesto.

