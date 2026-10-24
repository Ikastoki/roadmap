# NIO: Canales (Channels)

Mientras que `java.io` trabaja con flujos unidireccionales, NIO introduce el concepto de canal (`Channel`) como una abstraccion bidireccional que permite tanto lectura como escritura sobre una misma entidad. Un canal representa una conexion abierta hacia una entidad capaz de realizar operaciones de E/S, como un archivo, un socket o un dispositivo hardware. Los canales estan disenados para operar con `ByteBuffer`, no directamente con arreglos de bytes, lo que proporciona un control mas fino sobre la transferencia de datos.

La interfaz `Channel` define unicamente dos metodos: `isOpen()` para verificar si el canal esta abierto, y `close()` para cerrarlo. Todas las operaciones de transferencia de datos se definen en subinterfaces como `ReadableByteChannel`, `WritableByteChannel` e `InterruptibleChannel`. La creacion de canales se realiza mediante metodos static factory, tipicamente `open()`, no mediante constructores.

## FileChannel

`FileChannel` permite leer, escribir, modificar y truncar archivos, asi como transferir datos directamente entre canales sin pasar por memoria de usuario mediante los metodos `transferTo()` y `transferFrom()`. A diferencia de los flujos clasicos, `FileChannel` mantiene una posicion interna que puede consultarse con `position()` y modificarse con `position(long)`.

```java
RandomAccessFile file = new RandomAccessFile("datos.bin", "rw");
FileChannel channel = file.getChannel();

ByteBuffer buf = ByteBuffer.allocate(1024);
int bytesRead = channel.read(buf);

buf.flip();
String datos = StandardCharsets.UTF_8.decode(buf).toString();

buf.clear();
buf.put("nuevos datos".getBytes(StandardCharsets.UTF_8));
buf.flip();
channel.write(buf);

channel.close();
file.close();
```

Una caracteristica diferencial de `FileChannel` es que puede bloquear regiones del archivo para acceso exclusivo o compartido mediante `lock()` y `tryLock()`, devolviendo un `FileLock` que se libera al cerrar el canal o invocar `release()`.

## SocketChannel

`SocketChannel` es un canal orientado a conexion TCP que puede operar tanto en modo bloqueante como no bloqueante. Se crea mediante `SocketChannel.open()` y se conecta con `connect(SocketAddress)`. En modo bloqueante, `connect()` retorna unicamente cuando la conexion se establece o falla. En modo no bloqueante, `connect()` retorna inmediatamente y el programa debe verificar la conexion con `finishConnect()`.

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.configureBlocking(false);
socketChannel.connect(new InetSocketAddress("example.com", 80));

while (!socketChannel.finishConnect()) {
    // Esperar sin bloquear
}

ByteBuffer buffer = ByteBuffer.allocate(256);
buffer.put("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n".getBytes());
buffer.flip();
socketChannel.write(buffer);

buffer.clear();
socketChannel.read(buffer);
buffer.flip();
```

El metodo `configureBlocking(boolean)` es el mecanismo que permite alternar entre modo bloqueante (por defecto) y no bloqueante. En modo no bloqueante, `read()` retorna inmediatamente con los bytes disponibles (pueden ser cero) y `write()` escribe tantos bytes como pueda sin bloquear.

> En modo no bloqueante, `read()` puede retornar 0 si no hay datos disponibles, o -1 si el par remoto cerro la conexion. Es responsabilidad del programador distinguir entre ambos casos y actuar en consecuencia.

## ServerSocketChannel

`ServerSocketChannel` es el equivalente de `ServerSocket` pero orientado a canales. Escucha conexiones entrantes TCP en un puerto y acepta nuevas conexiones devolviendo un `SocketChannel`. Se configura con `bind(SocketAddress)` y se pone en modo no bloqueante con `configureBlocking(false)`.

```java
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.bind(new InetSocketAddress(8080));

while (true) {
    SocketChannel clientChannel = serverChannel.accept();
    if (clientChannel != null) {
        // Nueva conexion entrante
        clientChannel.configureBlocking(false);
        // Asignar a un Selector o procesar directamente
    }
}
```

En modo no bloqueante, `accept()` retorna inmediatamente: si hay una conexion pendiente devuelve un `SocketChannel`, si no hay conexiones pendientes devuelve `null`. Esto permite que un solo hilo pueda atender muchas conexiones entrantes sin bloquearse.

## DatagramChannel

`DatagramChannel` implementa comunicacion no orientada a conexion mediante UDP. A diferencia de `SocketChannel`, no utiliza `connect()` como paso obligatorio: puede enviar y recibir datagramas hacia cualquier destino sin establecer una conexion previa. Envia datagramas con `send(ByteBuffer, SocketAddress)` y recibe con `receive(ByteBuffer)`.

```java
DatagramChannel channel = DatagramChannel.open();
channel.configureBlocking(false);
channel.bind(new InetSocketAddress(9999));

ByteBuffer buffer = ByteBuffer.allocate(65507);
SocketAddress remoto = channel.receive(buffer);

if (remoto != null) {
    buffer.flip();
    System.out.println("Recibido de: " + remoto);
}
```

`DatagramChannel` tambien soporta el metodo `connect()` opcionalmente. Si se conecta a un destino, solo aceptara datagramas de ese origen y podra usar `read()` y `write()`, que son mas convenientes que `send()` y `receive()`.
