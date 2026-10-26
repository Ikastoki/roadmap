# Enrutamiento

El enrutamiento es el proceso por el cual un router determina la mejor ruta para enviar un paquete desde su origen hasta su destino a traves de una red interconectada. Los routers son los dispositivos que implementan el enrutamiento: mantienen tablas de enrutamiento que mapean redes de destino con los caminos para alcanzarlas, y toman decisiones paquete a paquete basandose en estas tablas.

El enrutamiento es la funcion que permite que internet funcione: sin routers, cada red local estaria aislada y no seria posible la comunicacion entre dispositivos en diferentes ciudades, paises o continentes.

## Tabla de enrutamiento

Cada router mantiene una **tabla de enrutamiento** que lista todas las redes alcanzables y el camino para llegar a ellas. La tabla contiene entradas como:

```
# ip route show
default via 192.168.1.1 dev eth0 proto dhcp metric 100
10.0.0.0/8 via 10.1.1.1 dev eth1 proto static metric 200
172.16.0.0/12 dev eth1 proto kernel scope link src 172.16.0.1
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

Cada entrada especifica:

- **Red de destino:** la red a la que se dirige el paquete.
- **Gateway (*via*):** la direccion IP del siguiente router en el camino.
- **Interfaz:** por que interfaz de red se debe enviar el paquete.
- **Metric:** el coste de la ruta (a menor metric, mejor ruta).
- **Proto:** el protocolo que origino la ruta (static, kernel, dhcp, ospf, etc.).

## Gateway predeterminado

El **gateway predeterminado** es el router al que se envian todos los paquetes cuyo destino no esta en la misma subred del emisor. Es la "puerta de salida" de la red local hacia internet.

Cuando un dispositivo quiere comunicarse con una IP fuera de su subred, envia el paquete al gateway predeterminado, que se encarga de reenviarlo a traves de la red hasta llegar a la red de destino.

```bash
# Ver el gateway predeterminado
ip route show default
route -n

# Configurar gateway estatico
sudo ip route add default via 192.168.1.1
```

## Enrutamiento estatico

El enrutamiento estatico utiliza rutas configuradas manualmente por el administrador. Es simple, predecible y no consume recursos de CPU ni ancho de banda para mantener las tablas actualizadas.

```bash
# Añadir ruta estatica
sudo ip route add 10.0.2.0/24 via 10.0.1.1

# Eliminar ruta
sudo ip route del 10.0.2.0/24
```

El enrutamiento estatico es adecuado para redes pequenas, enlaces punto a punto y rutas predeterminadas. No escala bien en redes grandes porque cada cambio de topologia requiere reconfiguracion manual.

## Enrutamiento dinamico

El enrutamiento dinamico utiliza protocolos que intercambian informacion entre routers para construir y mantener automaticamente las tablas de enrutamiento. Cuando un enlace falla, los routers recalculan las rutas de forma automatica sin intervencion humana.

**Protocolos de vector de distancia:**

- **RIP (*Routing Information Protocol*):** basado en hop count (maximo 15 saltos). Simple pero lento para converger y limitado a redes pequenas. Utiliza UDP como transporte.
- **EIGRP (*Enhanced Interior Gateway Routing Protocol*):** protocolo propietario de Cisco (ahora estandarizado como RFC). Mas avanzado que RIP, con convergencia rapida y metricas flexibles.

**Protocolos de estado de enlace:**

- **OSPF (*Open Shortest Path First*):** protocolo abierto que utiliza el algoritmo de Dijkstra para calcular las mejores rutas. Converge rapidamente, es escalable y soporta autenticacion. Es el protocolo de enrutamiento interno (IGP) mas utilizado en redes corporativas.
- **IS-IS (*Intermediate System to Intermediate System*):** similar a OSPF, utilizado en backbones de ISPs y redes de telecomunicaciones.

**Protocolos de path vector:**

- **BGP (*Border Gateway Protocol*):** el protocolo de enrutamiento externo (EGP) que conecta sistemas autonomos (AS) en internet. Es el protocolo que mantiene unida la red de redes, tomando decisiones de enrutamiento basadas en politicas, prefijos y atributos de camino.

## Estatico vs dinamico

| Caracteristica | Estatico | Dinamico |
|---|---|---|
| Configuracion | Manual | Automatica |
| Mantenimiento | Bajo | Medio-Alto |
| Convergencia | Instantanea | Variable (segundos a minutos) |
| Escalabilidad | Baja | Alta |
| Consumo de recursos | Ninguno | CPU, memoria, ancho de banda |
| Caso de uso | Redes pequenas, enlaces P2P | Redes grandes, ISPs |

> El enrutamiento es una de las areas mas complejas y criticas de las redes. Un error en la configuracion de rutas puede causar bucles, perdida de conectividad o trafico suboptimal. Los profesionales de redes deben dominar tanto el enrutamiento estatico (para configuraciones simples y de emergencia) como los protocolos dinamicos (para entornos productivos de cualquier tamano).

