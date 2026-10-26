# Tramas y control de acceso al medio

La capa de enlace de datos organiza los bits transmitidos por la capa fisica en unidades estructuradas llamadas **tramas** (*frames*). La trama es la PDU de la capa 2 e incluye cabeceras con informacion de control, los datos utiles (payload) y a veces una cola con checksums de integridad. Ademas, la capa de enlace determina que dispositivo puede transmitir en cada momento a traves de mecanismos de acceso al medio.

Esta capa resuelve tres problemas fundamentales: como estructurar los datos para su transmision, como compartir el medio entre multiples dispositivos y como detectar errores en la transmision.

## Estructura de una trama

Una trama tipica de Ethernet tiene la siguiente estructura:

```
┌─────────┬──────────┬──────────┬──────────┬─────┐
│Preambulo│MAC Orig  │MAC Dest  │Tipo/Len  │Datos│ FCS │
│ 8 bytes │ 6 bytes  │ 6 bytes  │ 2 bytes  │46-  │4byte│
│         │          │          │          │1500 │     │
└─────────┴──────────┴──────────┴──────────┴─────┘
```

- **Preambulo:** secuencia de bits que permite al receptor sincronizarse con el emisor (7 bytes) y delimitar el inicio de la trama (1 byte SFD).
- **MAC origen (6 bytes):** direccion MAC del emisor.
- **MAC destino (6 bytes):** direccion MAC del receptor (puede ser broadcast FF:FF:FF:FF:FF:FF).
- **Tipo/Longitud (2 bytes):** identifica el protocolo encapsulado (0x0800 = IPv4, 0x0806 = ARP) o la longitud de los datos.
- **Datos (payload):** el contenido util, entre 46 y 1500 bytes.
- **FCS (*Frame Check Sequence*):** checksum de 32 bits para detectar errores de transmision.

## Diferencia entre frame, packet y segment

Es comun confundir estos terminos pero se refieren a PDU de capas diferentes:

- **Frame (trama):** PDU de capa 2 (enlace de datos). Incluye direcciones MAC.
- **Packet (paquete):** PDU de capa 3 (red). Incluye direcciones IP.
- **Segment:** PDU de capa 4 (transporte). Incluye puertos.

Un paquete (IP) se encapsula dentro de una trama (Ethernet) para su transmision en la red local. Un segmento (TCP) se encapsula dentro de un paquete (IP).

## Metodos de acceso al medio

Cuando multiples dispositivos comparten el mismo canal de comunicacion, es necesario un mecanismo que determine quien puede transmitir en cada momento. Los metodos principales son:

### CSMA/CD (Ethernet clasico)

*Carrier Sense Multiple Access with Collision Detection* se utilizaba en Ethernet con topologia de bus (10BASE2, 10BASE5) y hub, donde todos los dispositivos compartian el mismo dominio de colision.

1. **Escuchar el medio:** antes de transmitir, el dispositivo verifica si el canal esta libre (*Carrier Sense*).
2. **Transmitir:** si el canal esta libre, comienza a enviar la trama.
3. **Detectar colision:** si otro dispositivo transmite simultaneamente, se produce una colision.
4. **Detener y enviar senal de interferencia (*jam signal*):** ambos dispositivos detienen la transmision y envian una senal para que todos los dispositivos detecten la colision.
5. **Reintentar:** cada dispositivo espera un tiempo aleatorio (*backoff*) y reintenta.

Con la llegada de los switches (que eliminan dominios de colision), CSMA/CD ha quedado praticamente obsoleto.

### CSMA/CA (Wi-Fi)

*Carrier Sense Multiple Access with Collision Avoidance* es el mecanismo utilizado en redes inalambricas. En Wi-Fi no es posible detectar colisiones durante la transmision (el dispositivo esta transmitiendo y no puede escuchar), por lo que se centra en prevenirlas.

1. **Escuchar el medio:** verificar si el canal esta libre.
2. **Espera aleatoria:** esperar un tiempo aleatorio para reducir la probabilidad de colision.
3. **Solicitar envio (*RTS/CTS*):** opcionalmente, enviar una solicitud de transmision y esperar confirmacion.
4. **Transmitir:** enviar la trama.
5. **Confirmar (*ACK*):** esperar un acuse de recibo del receptor. Si no llega, reintenta.

### Token Passing (Token Ring)

Un token (mensaje especial) circula por la red en una topologia de anillo. Solo el dispositivo que posee el token puede transmitir. Cuando termina, pasa el token al siguiente dispositivo. No hay colisiones porque solo un dispositivo transmite a la vez, pero la latencia puede ser elevada si hay muchos dispositivos.

## Flujo y control basico en capa 2

El control de flujo en capa 2 garantiza que el emisor no envie datos mas rapidos de los que el receptor puede procesar. Los mecanismos incluyen:

- **Stop-and-Wait:** el emisor envia una trama y espera el ACK antes de enviar la siguiente. Simple pero ineficiente.
- **Ventana deslizante:** el emisor puede enviar multipes tramas sin esperar ACK, siempre que no exceda el tamano de la ventana. Es el mecanismo utilizado en Ethernet moderno.

> El control de acceso al medio es fundamental en redes compartidas. Ethernet moderno con switches opera en full-duplex y no necesita mecanismos de acceso al medio (cada puerto es un dominio de colision independiente), pero en Wi-Fi sigue siendo critico porque el medio es compartido por naturaleza.

