# Topologias de red

La topologia de red define como estan organizados y conectados los dispositivos (nodos) en una red. La topologia determina el camino que siguen los datos, la fiabilidad de la comunicacion, la escalabilidad y el coste de la infraestructura. Elegir la topologia adecuada es una de las decisiones mas importantes en el diseno de una red.

Las topologias se clasifican en dos categorias fundamentales: topologia fisica (como estan cableados los dispositivos) y topologia logica (como viajan los datos independientemente del cableado). Una topologia fisica en estrella puede comportarse logicamente como un bus, y viceversa.

## Topologia fisica vs logica

**Topologia fisica:** describe la disposicion real de los cables, dispositivos y conexiones. Es lo que se ve cuando se observa el cableado de la red: donde estan los cables, como estan ramificados y donde estan los puntos de conexion.

**Topologia logica:** describe como viajan los datos a traves de la red, independientemente de la disposicion fisica del cableado. Determina que dispositivo recibe los datos y como se propagan por la red.

La distincion es importante porque la topologia fisica puede ser diferente de la logica. Por ejemplo, una red con topologia fisica en estrella (todos los dispositivos conectados a un switch central) puede operar logicamente como un bus si el switch actua como repetidor simple.

## Topologia en bus

En la topologia en bus, todos los dispositivos comparten un unico cable de comunicaciones (el bus). Los datos enviados por cualquier dispositivo recorren el bus en ambas direcciones y son recibidos por todos los dispositivos, pero solo el destinatario los procesa.

```
  A ──── B ──── C ──── D ──── E
  ───────────────────────────────  (bus compartido)
```

**Ventajas:** simplicidad, bajo coste de cableado, facil instalacion.

**Desventajas:** si el bus principal falla, toda la red cae; el rendimiento se degrada con muchos dispositivos por colisiones; dificil de diagnosticar fallos. Prcticamente en desuso en redes modernas.

## Topologia en estrella

Todos los dispositivos estan conectados a un punto central (switch, hub o router). Todos los datos pasan por el dispositivo central, que los reenvia al destinatario.

```
        A       B
         \     /
          [SW]
         /     \
        C       D
```

**Ventajas:** si un cable falla, solo un dispositivo queda desconectado; facil de instalar y mantener; escalable (añadir dispositivos es sencillo).

**Desventajas:** si el dispositivo central falla, toda la red cae; requiere mas cableado que la topologia en bus.

Es la topologia predominante en las redes locales modernas, tanto cableadas (Ethernet con switch) como inalambricas (Wi-Fi con punto de acceso).

## Topologia en anillo

Cada dispositivo esta conectado exactamente con dos vecinos, formando un circulo cerrado. Los datos viajan en una direccion (o en ambas en anillo dual) pasando de un nodo al siguiente hasta llegar al destinatario.

```
    A ─── B
    │     │
    D ─── C
```

**Ventajas:** no hay colisiones (cada nodo tiene su turno); rendimiento predecible.

**Desventajas:** si un nodo falla, la red puede quedar inutilizada (a menos que sea un anillo dual); añadir o eliminar dispositivos requiere interrumpir la red; complejidad de gestion.

Token Ring (IEEE 802.5) fue la implementacion mas conocida de esta topologia, pero ha sido completamente reemplazada por Ethernet.

## Topologia en malla

Cada dispositivo esta conectado con multiples (o todos) los demas dispositivos de la red. En una malla completa, cada nodo tiene una conexion directa con todos los demas; en una malla parcial, las conexiones son selectivas.

```
    A ─── B
    │ ╲ ╱ │
    │  ╳  │
    │ ╱ ╲ │
    D ─── C
```

**Ventajas:** alta redundancia y tolerancia a fallos (si un camino falla, hay alternativos); distribucion de carga entre multiples caminos.

**Desventajas:** coste elevado de cableado y configuracion; complejidad de administracion; no escalable a gran escala sin una malla parcial planificada.

Las mallas completas se utilizan en nucleos de centros de datos y en la infraestructura de internet backbone, donde la redundancia justifica el coste.

## Topologia en arbol

Es una combinacion de topologias en estrella con jerarquia. Los dispositivos se organizan en niveles: los nodos hoja se conectan a switches de nivel inferior, que a su vez se conectan a un switch de nivel superior, formando una estructura jerarquica tipo arbol.

```
           [Raiz]
          /      \
       [SW1]    [SW2]
       /  \      /  \
      A    B    C    D
```

**Ventajas:** escalable y facil de gestionar; aporta jerarquia a la red.

**Desventajas:** si un nodo padre falla, todos los nodos hijos quedan desconectados; la profundidad del arbol limita el rendimiento.

## Resumen comparativo

| Topologia | Fiabilidad | Coste | Escalabilidad | Uso comun |
|---|---|---|---|---|
| Bus | Baja | Muy bajo | Baja | Obsoleto |
| Estrella | Media | Medio | Alta | LAN modernas |
| Anillo | Media-Baja | Medio | Media | Obsoleto (Token Ring) |
| Malla | Alta | Alto | Media-Alta | Centros de datos, backbone |
| Arbol | Media-Alta | Medio-Alto | Alta | Redes corporativas jerarquicas |

> En la practica, las redes modernas utilizan topologias hibridas. Un centro de datos puede tener un nucleo en malla, un nivel de acceso en estrella y una distribucion en arbol. La topologia pura es rara; lo habitual es combinar elementos de varias topologias segun las necesidades de rendimiento, redundancia y coste.

