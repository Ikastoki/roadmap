# Funcion de la capa de transporte

La capa de transporte proporciona comunicacion de extremo a extremo entre procesos que se ejecutan en hosts diferentes. A diferencia de la capa de red, que direccione hosts usando direcciones IP, la capa de transporte direccione procesos individuales usando puertos como identificadores. Es la capa que separa la logica de red (envio de paquetes entre hosts) de la logica de aplicacion (intercambio de datos entre procesos).

La capa de transporte es donde se decide si la comunicacion debe ser confiable (TCP) o rapida (UDP). Esta decision tiene un impacto profundo en el rendimiento, la fiabilidad y la experiencia de usuario de las aplicaciones de red.

## Multiplexacion por puertos

La multiplexacion permite que multiples aplicaciones en un mismo host compartan una unica direccion IP. Cada aplicacion se identifica por un numero de puerto (16 bits, rango 0-65535), y la combinacion de IP + puerto + protocolo (TCP o UDP) identifica univocamente una conexion.

```
Host A (192.168.1.10)          Host B (10.0.0.5)
[Proceso Web :80] ←─────────→ [Proceso Web :80]
[Proceso SSH :22] ←─────────→ [Proceso SSH :22]
[Proceso DNS :53] ←─────────→ [Proceso DNS :53]
```

## Control de flujo

El control de flujo evita que un emisor rapido sature a un receptor lento. TCP implementa control de flujo con una **ventana de recepcion**: el receptor indica al emisor cuantos bytes puede enviar antes de recibir un acuse de recibo. Si la ventana llega a cero, el emisor detiene la transmision hasta que el receptor libere espacio y actualice la ventana.

## Fiabilidad

TCP proporciona fiabilidad mediante:

- **Numeros de secuencia:** cada segmento tiene un numero de secuencia que permite al receptor ordenar los segmentos y detectar perdidos.
- **Acuses de recibo (ACK):** el receptor confirma la recepcion correcta de datos.
- **Retransmision:** si el emisor no recibe ACK en un tiempo determinado (*timeout*), retransmite el segmento.
- **Deteccion de duplicados:** si el receptor recibe un segmento duplicado (por un ACK perdido), lo descarta.

UDP no proporciona ninguna de estas garantias: los datos se envian sin confirmar y sin retransmision. Si un datagrama UDP se pierde, se pierde definitivamente.

> La capa de transporte es el puente entre el mundo de las redes (paquetes, routers, direcciones) y el mundo de las aplicaciones (mensajes, archivos, flujos). TCP y UDP representan dos filosofias opuestas: fiabilidad a cambio de overhead (TCP) y minimalismo a cambio de velocidad (UDP). La mayoria de las aplicaciones se benefician de una de las dos, y la eleccion entre TCP y UDP es una de las decisiones mas importantes en el diseno de protocolos de red.

