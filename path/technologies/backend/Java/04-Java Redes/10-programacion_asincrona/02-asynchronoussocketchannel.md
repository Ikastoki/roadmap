# AsynchronousSocketChannel

`AsynchronousSocketChannel` pertenece al paquete `java.nio.channels` y proporciona E/S de red verdaderamente asincrona. A diferencia de `SocketChannel` en modo no bloqueante (que requiere sondeo explicito o un `Selector`), `AsynchronousSocketChannel` notifica la finalizacion de las operaciones mediante `Future` o `CompletionHandler`, liberando al programador de la gestion de bucles de eventos.

## Creacion del canal

El canal se obtiene mediante el metodo estatico `open()`, que puede utilizar el grupo de hilos por defecto o uno personalizado.

```java
// Grupo por defecto
AsynchronousSocketChannel ch = AsynchronousSocketChannel.open();

// Grupo personalizado
AsynchronousChannelGroup group = AsynchronousChannelGroup
    .withFixedThreadPool(10, Executors.defaultThreadFactory());
AsynchronousSocketChannel chCustom = AsynchronousSocketChannel.open(group);
```

El `AsynchronousChannelGroup` gestiona los hilos que procesan las operaciones de E/S. Por defecto, se crea un grupo interno con un pool basado en el numero de procesadores disponibles.

## Conexion asincrona

`connect()` acepta un `SocketAddress` y, opcionalmente, un `CompletionHandler`. Con la sobrecarga que devuelve `Future`, el control de flujo se delega al llamante.

```java
// Usando Future
AsynchronousSocketChannel ch = AsynchronousSocketChannel.open();
Future<Void> future = ch.connect(new InetSocketAddress("example.com", 80));
Void result = future.get(); // bloquea hasta conectar
```

```java
// Usando CompletionHandler (no bloqueante)
ch.connect(new InetSocketAddress("example.com", 80), null, new CompletionHandler<Void, Void>() {
    @Override
    public void completed(Void result, Void attachment) {
        System.out.println("Conectado exitosamente");
        iniciarLectura(ch);
    }

    @Override
    public void failed(Throwable exc, Void attachment) {
        System.err.println("Error de conexion: " + exc.getMessage());
    }
});
```

El segundo parametro del `CompletionHandler` es un `attachment` que el desarrollador puede usar para pasar contexto (como el buffer o el estado de la sesion). Puede ser cualquier objeto o `null`.

## Lectura asincrona

`read()` funciona de manera similar, devolviendo un `Future<Integer>` (numero de bytes leidos) o usando un `CompletionHandler`.

```java
ByteBuffer buf = ByteBuffer.allocate(4096);

// Con Future
Future<Integer> readFuture = ch.read(buf);
int bytesRead = readFuture.get();
```

```java
// Con CompletionHandler
ch.read(buf, buf, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer buffer) {
        if (result == -1) {
            System.out.println("Conexion cerrada por el remoto");
            cerrarConexion();
            return;
        }
        buffer.flip();
        byte[] data = new byte[buffer.remaining()];
        buffer.get(data);
        System.out.println("Recibido: " + new String(data, StandardCharsets.UTF_8));
        buffer.clear();
        // Continuar leyendo...
        ch.read(buffer, buffer, this);
    }

    @Override
    public void failed(Throwable exc, ByteBuffer buffer) {
        System.err.println("Error de lectura: " + exc.getMessage());
    }
});
```

Notese como el `CompletionHandler` se pasa a si mismo (`this`) en la siguiente llamada a `read()`, estableciendo una cadena de lecturas asincronas sin bloquear ningun hilo.

## Escritura asincrona

`write()` sigue el mismo patron. Tambien devuelve `Future<Integer>` o utiliza `CompletionHandler`.

```java
ByteBuffer out = ByteBuffer.wrap("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n".getBytes());

ch.write(out, out, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer buffer) {
        if (buffer.hasRemaining()) {
            // Aun no se ha enviado todo
            ch.write(buffer, buffer, this);
        } else {
            System.out.println("Peticion enviada completamente");
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer buffer) {
        System.err.println("Error de escritura: " + exc.getMessage());
    }
});
```

## Cierre

`close()` cierra el canal. Las operaciones pendientes se completan con `AsynchronousCloseException` o `ClosedChannelException`.

```java
ch.close(); // libera todos los recursos asociados
```

## Ejemplo completo: cliente HTTP asincrono

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.StandardCharsets;

public class ClienteHTTPAsync {
    public static void main(String[] args) throws Exception {
        AsynchronousSocketChannel ch = AsynchronousSocketChannel.open();
        ch.connect(new InetSocketAddress("jsonplaceholder.typicode.com", 80), null,
            new CompletionHandler<Void, Void>() {
                @Override
                public void completed(Void result, Void attach) {
                    String request = "GET /posts/1 HTTP/1.1\r\n"
                        + "Host: jsonplaceholder.typicode.com\r\n"
                        + "Connection: close\r\n\r\n";
                    ByteBuffer buf = ByteBuffer.wrap(request.getBytes());
                    ch.write(buf, buf, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer written, ByteBuffer buffer) {
                            if (buffer.hasRemaining()) {
                                ch.write(buffer, buffer, this);
                                return;
                            }
                            ByteBuffer response = ByteBuffer.allocate(8192);
                            ch.read(response, response, new CompletionHandler<Integer, ByteBuffer>() {
                                @Override
                                public void completed(Integer bytes, ByteBuffer rb) {
                                    if (bytes == -1) {
                                        rb.flip();
                                        System.out.println("Respuesta:");
                                        System.out.println(StandardCharsets.UTF_8.decode(rb));
                                        try { ch.close(); } catch (Exception e) {}
                                        return;
                                    }
                                    rb.clear();
                                    ch.read(rb, rb, this);
                                }
                                @Override
                                public void failed(Throwable ex, ByteBuffer rb) {
                                    System.err.println("Error lectura: " + ex.getMessage());
                                }
                            });
                        }
                        @Override
                        public void failed(Throwable ex, ByteBuffer buffer) {
                            System.err.println("Error escritura: " + ex.getMessage());
                        }
                    });
                }
                @Override
                public void failed(Throwable ex, Void attach) {
                    System.err.println("Error conexion: " + ex.getMessage());
                }
            });
        // El hilo principal no se bloquea, pero debe mantenerse vivo
        Thread.sleep(5000);
    }
}
```

> El hilo principal en el ejemplo anterior necesita `Thread.sleep()` porque los hilos del grupo de E/S son daemon y la JVM terminaria inmediatamente. En aplicaciones reales, el bucle principal de la aplicacion o un framework (Spring, Vert.x) mantienen la JVM en ejecucion.
