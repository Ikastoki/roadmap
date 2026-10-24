# SocketChannel

`SocketChannel` es el equivalente no bloqueante del socket TCP clasico (`Socket`) dentro del paquete `java.nio.channels`. A diferencia del modelo de E/S de Java tradicional, `SocketChannel` opera sobre canales y buffers, y puede configurarse tanto en modo bloqueante como no bloqueante, ofreciendo el control necesario para construir aplicaciones de red escalables sin la sobrecarga de un hilo por conexion.

## Creacion y apertura

`SocketChannel` no tiene constructores publicos. Se obtiene mediante el metodo estatico `open()`, que puede abrir un canal sin conectar o conectarse directamente a un destino remoto.

```java
// Sin conectar (para conexion asincrona posterior)
SocketChannel ch = SocketChannel.open();

// Conectar directamente
SocketChannel ch = SocketChannel.open(new InetSocketAddress("example.com", 80));
```

## Configuracion de modo bloqueante

El metodo `configureBlocking(boolean block)` determina si las operaciones de E/S sobre el canal seran bloqueantes o no. Por defecto, un `SocketChannel` recien creado es bloqueante.

```java
SocketChannel ch = SocketChannel.open();
ch.configureBlocking(false); // modo no bloqueante
```

En modo no bloqueante, las operaciones de lectura/escritura retornan inmediatamente con los datos disponibles, sin esperar a que se complete la transferencia completa. Esto permite manejar multiples canales desde un unico hilo mediante un `Selector`.

## Conexion en modo no bloqueante

Cuando el canal esta en modo no bloqueante, `connect()` inicia el handshake TCP y retorna inmediatamente. La conexion probablemente no se haya completado aun; hay que verificar con `finishConnect()`.

```java
SocketChannel ch = SocketChannel.open();
ch.configureBlocking(false);
ch.connect(new InetSocketAddress("example.com", 80));

// Hacer otras tareas mientras se establece la conexion...

while (!ch.finishConnect()) {
    // esperar sin bloquear todo el hilo
    Thread.yield(); // en la practica se usa un Selector
}
```

`finishConnect()` debe llamarse repetidamente hasta que devuelva `true`. En una aplicacion real, esto se maneja registrando el canal en un `Selector` para la operacion de conexion (`SelectionKey.OP_CONNECT`).

```java
ch.configureBlocking(false);
ch.connect(new InetSocketAddress("host", 80));
Selector selector = Selector.open();
ch.register(selector, SelectionKey.OP_CONNECT);

while (!ch.finishConnect()) {
    selector.select(1000);
    // procesar eventos
}
```

## Lectura y escritura

Ambas operaciones trabajan con `ByteBuffer`. Al ser no bloqueantes, `read()` devuelve el numero de bytes leidos (pudiendo ser 0 o -1 si el canal ha sido cerrado por el remoto), y `write()` devuelve el numero de bytes escritos.

```java
ByteBuffer buf = ByteBuffer.allocate(4096);
int bytesRead = ch.read(buf); // en modo no bloqueante: 0, >0 o -1

if (bytesRead > 0) {
    buf.flip();
    byte[] data = new byte[buf.remaining()];
    buf.get(data);
    System.out.println("Recibido: " + new String(data, StandardCharsets.UTF_8));
    buf.clear();
} else if (bytesRead == -1) {
    System.out.println("El remoto cerro la conexion");
    ch.close();
}
```

Para escritura:

```java
ByteBuffer outBuf = ByteBuffer.wrap("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n".getBytes());
while (outBuf.hasRemaining()) {
    ch.write(outBuf);
}
```

En modo no bloqueante, write() puede no enviar todo el contenido en una unica llamada. El bucle asegura que se envien todos los bytes, aunque en la practica se obtiene el interese de escritura del `Selector` cuando el buffer tiene datos pendientes.

## Cierre

`close()` cierra el canal. Si el registro en un `Selector` aun esta activo, la clave asociada se cancelara automaticamente y se agregara al conjunto de claves canceladas del selector.

```java
ch.close(); // libera el socket subyacente
```

> En modo no bloqueante, `close()` puede lanzar `ClosedChannelException` en cualquier otra operacion pendiente sobre el canal. Es recomendable verificar `isConnected()` y `isOpen()` antes de realizar operaciones de E/S en codigo defensivo.

## isConnected y estado del canal

```java
System.out.println("Open: " + ch.isOpen());
System.out.println("Connected: " + ch.isConnected());
System.out.println("Connection pending: " + ch.isConnectionPending());
```

`isConnectionPending()` indica si se ha invocado `connect()` pero aun no se ha completado con `finishConnect()`. Es util en maquinas de estado para determinar la fase actual de la conexion.

## Ejemplo completo: cliente eco en modo no bloqueante

```java
SocketChannel ch = SocketChannel.open();
ch.configureBlocking(false);
ch.connect(new InetSocketAddress("localhost", 5000));

while (!ch.finishConnect()) {
    // espera activa o selector
}

ByteBuffer buf = ByteBuffer.wrap("Hola servidor".getBytes(StandardCharsets.UTF_8));
while (buf.hasRemaining()) {
    ch.write(buf);
}

ByteBuffer response = ByteBuffer.allocate(256);
ch.read(response);
response.flip();
byte[] data = new byte[response.remaining()];
response.get(data);
System.out.println("Eco: " + new String(data, StandardCharsets.UTF_8));
ch.close();
```
