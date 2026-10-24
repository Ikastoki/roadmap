# DatagramChannel

`DatagramChannel` es el canal NIO para comunicacion mediante UDP (User Datagram Protocol), un protocolo no orientado a conexion que intercambia datagramas sin garantias de entrega, orden o integridad. A diferencia de TCP, UDP no mantiene estado de conexion, lo que lo hace ideal para aplicaciones que priorizan la latencia sobre la fiabilidad, como streaming, DNS, juegos en red o telemetria.

## Apertura y configuracion

`DatagramChannel` se crea mediante su metodo estatico `open()`. Al igual que los demas canales NIO, puede configurarse como bloqueante o no bloqueante segun las necesidades de la aplicacion.

```java
DatagramChannel ch = DatagramChannel.open();
ch.configureBlocking(false);
ch.bind(new InetSocketAddress("0.0.0.0", 5000));
```

`bind()` asocia el canal a un puerto local. Sin `bind()`, el canal puede enviar datagramas pero no recibirlos; el sistema asigna un puerto efimero automaticamente.

## Envio de datagramas

`send()` entrega un datagrama a una direccion remota especifica. Acepta un `ByteBuffer` con los datos y un `SocketAddress` con el destino.

```java
DatagramChannel ch = DatagramChannel.open();
ch.configureBlocking(false);

ByteBuffer buf = ByteBuffer.wrap("Hola UDP".getBytes(StandardCharsets.UTF_8));
SocketAddress destino = new InetSocketAddress("192.168.1.10", 5000);
ch.send(buf, destino);
```

En modo no bloqueante, `send()` retorna inmediatamente indicando el numero de bytes puestos en el buffer de salida del socket. No hay garantia de que el datagrama llegue a su destino; la capa de transporte UDP no proporciona confirmacion de entrega.

## Recepcion de datagramas

`receive()` lee un datagrama entrante. En modo no bloqueante retorna `null` si no hay datos disponibles. Devuelve la direccion del remitente, permitiendo responder al origen.

```java
ByteBuffer buf = ByteBuffer.allocate(65507); // maximo tamano UDP
SocketAddress remitente = ch.receive(buf);

if (remitente != null) {
    buf.flip();
    byte[] data = new byte[buf.remaining()];
    buf.get(data);
    System.out.println("De " + remitente + ": " + new String(data, StandardCharsets.UTF_8));
}
```

El buffer debe ser suficientemente grande para contener el datagrama completo. El maximo teorico de un datagrama UDP es 65535 bytes, pero el limite practico suele ser 65507 bytes tras descontar las cabeceras IP y UDP. Si el buffer es menor que el datagrama, los bytes restantes se descartan silenciosamente.

## Conexion virtual

Aunque UDP no tiene conexion, `DatagramChannel.connect()` establece una conexion virtual que restringe el canal a un unico destino. Una vez conectado, pueden usarse `read()` y `write()` en lugar de `send()` y `receive()`, simplificando el codigo.

```java
DatagramChannel ch = DatagramChannel.open();
ch.configureBlocking(false);
ch.connect(new InetSocketAddress("ntp.example.com", 123));

// Ahora se puede usar read/write
ByteBuffer buf = ByteBuffer.wrap(new byte[48]); // peticion NTP
ch.write(buf);
buf.clear();
ch.read(buf);
```

`connect()` no intercambia paquetes en la red; es puramente local. Despues de conectar, solo se aceptan datagramas del destino especificado, y cualquier datagrama de otro origen se descarta. Para deshacer la conexion virtual se usa `disconnect()`.

## Desconexion

`disconnect()` revierte el efecto de `connect()`, dejando el canal listo para comunicarse con cualquier destino nuevamente.

```java
ch.disconnect(); // vuelve a modo no conectado
```

## Ejemplo completo: servidor eco UDP no bloqueante

```java
import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;
import java.nio.charset.StandardCharsets;

public class ServidorUDP {
    public static void main(String[] args) throws Exception {
        DatagramChannel ch = DatagramChannel.open();
        ch.configureBlocking(false);
        ch.bind(new InetSocketAddress(6000));
        System.out.println("Servidor UDP escuchando en puerto 6000");

        ByteBuffer buf = ByteBuffer.allocate(65507);

        while (true) {
            buf.clear();
            SocketAddress remitente = ch.receive(buf);
            if (remitente != null) {
                buf.flip();
                String msg = StandardCharsets.UTF_8.decode(buf).toString().trim();
                System.out.println("Recibido de " + remitente + ": " + msg);
                // Reenviar el mismo mensaje como eco
                buf.flip();
                ch.send(buf, remitente);
            }
            // Realizar otras tareas...
        }
    }
}
```

## Registro con Selector

`DatagramChannel` tambien se registra en un `Selector`, permitiendo que un unico hilo geste multiples canales UDP simultaneamente.

```java
Selector selector = Selector.open();
DatagramChannel ch = DatagramChannel.open();
ch.configureBlocking(false);
ch.bind(new InetSocketAddress(6000));
ch.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select();
    for (SelectionKey key : selector.selectedKeys()) {
        selector.selectedKeys().clear();
        if (key.isReadable()) {
            DatagramChannel dc = (DatagramChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(65507);
            SocketAddress remitente = dc.receive(buf);
            buf.flip();
            // procesar...
        }
    }
}
```

> La capacidad maxima de un datagrama UDP esta limitada por la MTU de la red (tipicamente 1500 bytes para Ethernet). Para mensajes mayores de 512 bytes, es recomendable implementar fragmentacion a nivel de aplicacion o usar TCP. `DatagramChannel` no fragmenta automaticamente; si se envia un buffer mayor que la MTU, el datagrama se fragmentara en IP con posible perdida de rendimiento.
