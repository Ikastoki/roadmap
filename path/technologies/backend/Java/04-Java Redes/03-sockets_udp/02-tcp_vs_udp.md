# Diferencias TCP vs UDP en Java

La eleccion entre TCP y UDP determina las caracteristicas fundamentales del transporte: fiabilidad, ordenacion, control de flujo y modelo de datos. TCP esta disenado para la precision, UDP para la velocidad. Cada protocolo resuelve un conjunto diferente de restricciones y su eleccion debe basarse en los requisitos de la aplicacion y no en la familiaridad con uno u otro.

## TCP: orientado a conexion y confiable

TCP establece una conexion mediante el three-way handshake antes de transmitir datos. Este intercambio inicial (SYN, SYN-ACK, ACK) negocia numeros de secuencia, ventana de recepcion y opciones como MSS (Maximum Segment Size).

```java
// TCP: establece conexion antes de enviar datos
Socket socket = new Socket("host", 8080);
OutputStream out = socket.getOutputStream();
out.write("datos".getBytes());
```

TCP garantiza que todos los bytes enviados llegan al receptor en el mismo orden, sin duplicados y sin perdidas. Si un segmento se pierde, TCP lo retransmite automaticamente. Si llega desordenado, TCP lo reordena antes de entregarlo a la aplicacion. El receptor envia acuses de recibo (ACK) acumulativos que confirman la recepcion de todos los bytes hasta cierto punto.

El control de congestion implementado en TCP (Cubic, BBR, Reno) ajusta la velocidad de transmision para evitar saturar la red. Cuando se detecta perdida de paquetes, TCP reduce la ventana de congestion y retransmite, sacrificando rendimiento momentaneo para preservar la estabilidad de la red.

Desde la perspectiva de la API de Java, TCP se presenta como un flujo continuo de bytes (`InputStream`/`OutputStream`). La aplicacion no tiene nocion de donde empieza o termina un mensaje: debe implementar su propio protocolo de delimitacion (lineas con `\n`, prefijos de longitud, o formatos binarios con campos de tamano fijo).

## UDP: sin conexion y no confiable

UDP no establece conexion. El emisor construye un datagrama con los datos, la direccion de destino y el puerto, y lo entrega a la capa IP para su transmision. No hay garantia de que el datagrama llegue a su destino, y si llegan varios, pueden hacerlo en cualquier orden.

```java
// UDP: envia sin establecer conexion previa
DatagramSocket socket = new DatagramSocket();
byte[] datos = "datos".getBytes();
DatagramPacket packet = new DatagramPacket(datos, datos.length,
        InetAddress.getByName("host"), 8080);
socket.send(packet);
```

Si un datagrama se pierde, UDP no lo retransmite. Si llega corrupto, UDP lo descarta (el checksum detecta errores pero no los corrige). La aplicacion debe manejar estas condiciones si las necesita.

El tamano maximo de un datagrama UDP es de 65507 bytes de datos (65535 menos 20 de IP y 8 de UDP). En la practica, los datagramas que superan la MTU de la red (tipicamente 1500 bytes) se fragmentan a nivel IP, y la perdida de un solo fragmento descarta el datagrama completo.

La API de Java refleja esta naturaleza sin estado: `DatagramSocket.send()` lanza el datagrama a la red sin saber si llegara, y `DatagramSocket.receive()` recibe datagramas de cualquier remitente. No hay concepto de conexion, solo de paquetes individuales intercambiados entre puertos.

## Diferencias clave

| Aspecto | TCP | UDP |
|---|---|---|
| Conexion | Orientado a conexion | Sin conexion |
| Fiabilidad | Garantiza entrega | No garantiza |
| Orden | Orden garantizado | Sin orden |
| Control de congestion | Si | No |
| Overhead | Alto (20 bytes cabecera + ACKs) | Bajo (8 bytes cabecera) |
| Modelo datos | Flujo continuo de bytes | Datagramas discretos |
| Velocidad | Menor (por ACKs y control) | Mayor |
| Estado | El socket mantiene estado | Sin estado |

## Casos de uso

TCP es la opcion correcta cuando la integridad de los datos es critica y se puede tolerar cierta latencia adicional. La web (HTTP/HTTPS), el correo electronico (SMTP, IMAP), la transferencia de archivos (FTP, SFTP) y las bases de datos relacionales son aplicaciones que se construyen sobre TCP.

UDP destaca en aplicaciones donde la puntualidad importa mas que la precision absoluta:

- Streaming de audio y video en vivo: perder un frame es aceptable; detenerse a retransmitirlo romperia la reproduccion.
- Juegos en red multijugador: la posicion de los jugadores se actualiza constantemente; un paquete perdido se reemplaza con la siguiente actualizacion.
- DNS: una consulta de resolucion de nombres cabe en un unico datagrama; si se pierde, el cliente simplemente reintenta.
- Voip y videoconferencia: el retardo maximo tolerable es de 150 ms; la retransmision TCP lo superaria.

## MulticastSocket

UDP soporta multicast, un modelo de comunicacion uno-a-muchos donde un unico datagrama se entrega a todos los miembros de un grupo. `MulticastSocket` extiende `DatagramSocket` para unirse a grupos multicast.

```java
MulticastSocket socket = new MulticastSocket(9092);
InetAddress grupo = InetAddress.getByName("230.0.0.1");
socket.joinGroup(grupo);

byte[] buffer = new byte[65507];
DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
socket.receive(packet); // recibe datagramas enviados al grupo
```

```java
// Emisor
MulticastSocket socket = new MulticastSocket();
InetAddress grupo = InetAddress.getByName("230.0.0.1");
String msg = "Anuncio a todos los miembros";
DatagramPacket packet = new DatagramPacket(
    msg.getBytes(), msg.length(), grupo, 9092);
socket.send(packet);
```

Las direcciones multicast estan en el rango 224.0.0.0 a 239.255.255.255. Los routers deben estar configurados para soportar multicast (IGMP) y los switches deben reenviar el trafico a los puertos apropiados. En redes locales sin configuracion especial, multicast funciona como broadcast limitado al segmento de red.

> TCP no soporta multicast directamente porque su modelo orientado a conexion requiere un canal punto a punto con estado. Para distribuir datos de forma confiable a multiples destinatarios se necesita una capa superior (por ejemplo, TCP con un proxy por destinatario, o protocolos como QUIC que incorporan multicast a nivel de aplicacion).
