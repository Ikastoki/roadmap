# MAC y LLC

La capa de enlace de datos se subdivide en dos subcapas que tienen responsabilidades diferentes: MAC (*Media Access Control*) y LLC (*Logical Link Control*). Esta division permite que la capa de enlace se adapte a diferentes tecnologias de capa fisica (Ethernet, Wi-Fi, PPP) manteniendo una interfaz comun con la capa de red.

La subcapa MAC es responsable del acceso fisico al medio y las direcciones hardware, mientras que LLC proporciona una interfaz independiente del hardware hacia la capa de red, gestionando multiplexacion de protocolos y control de errores basico.

## MAC (Media Access Control)

La subcapa MAC controla como los dispositivos acceden al medio fisico para transmitir. Sus responsabilidades incluyen:

- **Direccionamiento fisico:** asignacion y gestion de direcciones MAC, identificadores unicos de 48 bits grabados en la tarjeta de red (NIC).
- **Control de acceso al medio:** implementacion de mecanismos como CSMA/CD (Ethernet) o CSMA/CA (Wi-Fi) que determinan cuando puede transmitir un dispositivo.
- **Control de errores a nivel de trama:** verificacion del FCS (*Frame Check Sequence*) para detectar tramas corrompidas.
- **Delimitacion de tramas:** identificacion del inicio y fin de cada trama.

### Formato de una direccion MAC

Una direccion MAC es una secuencia de 6 bytes (48 bits) escrita tipicamente en notacion hexadecimal:

```
Ejemplo: 00:1A:2B:3C:4D:5E
         ││     ││     ││
         Fabricante(3 bytes)  ← OUI (Organizationally Unique Identifier)
                    ID unico (3 bytes) ← asignado por el fabricante
```

Los primeros 3 bytes identifican al fabricante (OUI, asignado por IEEE) y los 3 ultimos son un identificador unico asignado por el fabricante. Esto garantiza que cada tarjeta de red del mundo tiene una direccion MAC unica.

Las direcciones MAC pueden ser:

- **Unicast:** direciona a un dispositivo especifico (el bit menos significativo del primer byte es 0).
- **Broadcast:** FF:FF:FF:FF:FF:FF, se envia a todos los dispositivos de la red local.
- **Multicast:** direciona a un grupo de dispositivos (el bit menos significativo del primer byte es 1).

### Diferencia entre MAC e IP

| Caracteristica | MAC | IP |
|---|---|---|
| Capa | 2 (Enlace de datos) | 3 (Red) |
| Longitud | 48 bits (6 bytes) | 32 bits (IPv4) o 128 bits (IPv6) |
| Alcance | Red local (LAN) | Global (internet) |
| Asignacion | Fija por fabricante | Configurable / dinamica |
| Modificable | No (en teoria; puede ser modificada por software) | Si (estatica o DHCP) |
| Unicidad | Global (garantizada por IEEE) | Dentro de la red (puede repetirse en distintas redes) |

La MAC identifica un dispositivo en la red local; la IP identifica un dispositivo en la red global. Cuando un paquete IP cruza un router, la MAC se reemplaza pero la IP permanece igual.

## LLC (Logical Link Control)

La subcapa LLC proporciona una interfaz uniforme hacia la capa de red, independientemente de la tecnologia de capa fisica. Sus funciones incluyen:

- **Multiplexacion de protocolos:** permite que multiples protocolos de capa 3 (IPv4, IPv6, ARP) compartan el mismo enlace de datos. Utiliza un campo *EtherType* (o DSAP/SSAP en LLC puro) para identificar que protocolo esta encapsulado.
- **Control de errores basico:** proporciona un mecanismo de确认 de entrega opcional (no utilizado en Ethernet, que delega esta funcion a capas superiores).
- **Control de flujo basico:** mecanismos para evitar que un emisor rapido sature a un receptor lento.

En la practica moderna, LLC ha perdido relevancia porque Ethernet incorpora directamente la multiplexion en su campo *EtherType* y delega el control de flujo y errores a TCP (capa 4). Sin embargo, el concepto sigue siendo util para entender la arquitectura de la capa 2.

> La distincion MAC/LLC es un ejemplo de como la modularidad facilita el diseno: Ethernet, Wi-Fi y PPP tienen subcapas MAC muy diferentes pero comparten una interfaz LLC comun, lo que permite que la capa de red (IP) opere sin conocer los detalles del hardware subyacente.

