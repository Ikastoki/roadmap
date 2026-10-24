# DatagramSocket y DatagramPacket

UDP (User Datagram Protocol) ofrece un servicio de transporte sin conexion, no confiable y sin orden garantizado. A diferencia de TCP, no hay handshake, no hay reenvio de paquetes perdidos, no hay control de congestion y no hay garantia de que los datagramas lleguen en el mismo orden en que se enviaron. Lo que UDP pierde en fiabilidad lo gana en baja latencia y simplicidad, siendo la opcion preferida para streaming, juegos en red, DNS y protocolos donde la velocidad prevalece sobre la integridad absoluta.

## DatagramSocket

`DatagramSocket` es la clase principal para enviar y recibir datagramas UDP. Un datagrama es un mensaje autocontenido que incluye los datos, la direccion de destino y el puerto.

### Creacion

```java
DatagramSocket socket = new DatagramSocket();           // Puerto efimero
DatagramSocket socket = new DatagramSocket(9090);       // Puerto fijo
DatagramSocket socket = new DatagramSocket(9090, InetAddress.getByName("0.0.0.0")); // Puerto y direccion
```

El constructor sin argumentos vincula el socket a un puerto disponible asignado por el sistema (puerto efimero), util para clientes. El constructor con puerto es tipico de servidores que deben escuchar en una direccion conocida.

### Envio

Para enviar un datagrama se construye un `DatagramPacket` con los datos, la longitud, la direccion IP de destino y el puerto. Los datos se colocan en un array de bytes.

```java
String mensaje = "Hola UDP";
byte[] buffer = mensaje.getBytes(StandardCharsets.UTF_8);
InetAddress direccion = InetAddress.getByName("192.168.1.10");

DatagramPacket packet = new DatagramPacket(buffer, buffer.length, direccion, 9090);
socket.send(packet);
```

### Recepcion

Para recibir, se crea un `DatagramPacket` con un buffer vacio y se pasa a `receive()`. El metodo bloquea hasta que llega un datagrama y rellena el buffer con los datos recibidos. La direccion y puerto del remitente quedan almacenados en el mismo packet.

```java
byte[] buffer = new byte[65507];
DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
socket.receive(packet);

String mensaje = new String(packet.getData(), 0, packet.getLength(), StandardCharsets.UTF_8);
InetAddress address = packet.getAddress();
int port = packet.getPort();
```

## DatagramPacket

`DatagramPacket` encapsula tres conceptos: el contenido del mensaje (buffer de bytes), la metainformacion de enrutamiento (direccion y puerto) y la longitud efectiva de los datos recibidos.

### Constructores principales

- `DatagramPacket(byte[] buf, int length)`: para recepcion. `length` indica el tamano maximo aceptado.
- `DatagramPacket(byte[] buf, int length, InetAddress address, int port)`: para envio.
- `DatagramPacket(byte[] buf, int offset, int length)`: para recepcion con offset en el buffer.
- `DatagramPacket(byte[] buf, int offset, int length, InetAddress address, int port)`: para envio con offset.

### Metodos clave

Despues de una recepcion, los metodos de `DatagramPacket` revelan los detalles del datagrama entrante:

- `getData()`: devuelve el array de bytes completo (no solo los datos utiles).
- `getOffset()`: indice donde comienzan los datos utiles dentro del array.
- `getLength()`: numero de bytes recibidos realmente.
- `getAddress()`: direccion IP del remitente.
- `getPort()`: puerto del remitente.

> El buffer de recepcion debe ser lo suficientemente grande para albergar el datagrama entrante. Si el datagrama excede el tamano del buffer, el exceso se descarta silenciosamente. El tamano maximo teorico de un datagrama UDP es 65535 bytes, de los cuales 8 son para la cabecera UDP y 20 para la cabecera IP, dejando 65507 bytes de datos utiles como maximo. En la practica, muchos sistemas limitan los datagramas a 1500 bytes (MTU de Ethernet) para evitar fragmentacion.

## Ejemplo: servidor UDP

El siguiente ejemplo implementa un servidor UDP que recibe mensajes en el puerto 9091 y los imprime en consola junto con la direccion del remitente.

```java
import java.net.*;
import java.nio.charset.StandardCharsets;

public class UDPServer {

    public static void main(String[] args) {
        int port = 9091;

        try (DatagramSocket socket = new DatagramSocket(port)) {
            System.out.println("Servidor UDP escuchando en puerto " + port);

            byte[] buffer = new byte[65507];

            while (true) {
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
                socket.receive(packet);

                String mensaje = new String(packet.getData(), 0, packet.getLength(),
                        StandardCharsets.UTF_8);
                String remitente = packet.getAddress().getHostAddress() + ":" + packet.getPort();

                System.out.println("Recibido de " + remitente + ": " + mensaje);

                // Opcional: enviar respuesta
                String respuesta = "Recibido: " + mensaje;
                byte[] respBytes = respuesta.getBytes(StandardCharsets.UTF_8);
                DatagramPacket respPacket = new DatagramPacket(
                        respBytes, respBytes.length,
                        packet.getAddress(), packet.getPort());
                socket.send(respPacket);
            }

        } catch (IOException e) {
            System.err.println("Error en servidor UDP: " + e.getMessage());
        }
    }
}
```

El servidor entra en un bucle infinito de recepcion. Por cada datagrama entrante, imprime el contenido y opcionalmente responde al remitente. No hay conexion que mantener, no hay hilos por cliente: un unico hilo puede atender a todos los remitentes secuencialmente.

> El uso de `try-with-resources` en `DatagramSocket` cierra el socket al salir del bloque. El cierre libera el puerto en el sistema operativo. A diferencia de TCP, no hay intercambio de mensajes de cierre: UDP simplemente deja de escuchar.
