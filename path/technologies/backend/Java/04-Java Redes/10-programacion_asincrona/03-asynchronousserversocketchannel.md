# AsynchronousServerSocketChannel

`AsynchronousServerSocketChannel` es el canal servidor para E/S asincrona en Java NIO.2. Su funcion es aceptar conexiones TCP entrantes sin bloquear el hilo de llamada, notificando la llegada de nuevas conexiones mediante `CompletionHandler`. Forma parte del mismo modelo asincrono que `AsynchronousSocketChannel` y comparte el grupo de hilos asociado.

## Apertura y configuracion

Se crea con `open()` y se asocia a un grupo de canales asincronos (opcional). `bind()` lo vincula a una direccion y puerto local.

```java
AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open();
server.bind(new InetSocketAddress("0.0.0.0", 9000));
```

Para usar un grupo personalizado:

```java
AsynchronousChannelGroup group = AsynchronousChannelGroup
    .withFixedThreadPool(10, Executors.defaultThreadFactory());
AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open(group);
server.bind(new InetSocketAddress(9000));
```

Sin grupo explicito, se utiliza el grupo por defecto de la JVM, compartido entre todos los canales asincronos.

## Accept con CompletionHandler

`accept()` recibe un `CompletionHandler<AsynchronousSocketChannel, A>` que se invoca cuando una conexion entrante es aceptada. El handler puede ser reutilizado para aceptar la siguiente conexion, creando un bucle asincrono.

```java
server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
    @Override
    public void completed(AsynchronousSocketChannel client, Void attachment) {
        // Aceptar la siguiente conexion inmediatamente
        server.accept(null, this);

        System.out.println("Cliente conectado: " + client.getRemoteAddress());
        // Iniciar lectura/escritura con el cliente
        leerCliente(client);
    }

    @Override
    public void failed(Throwable exc, Void attachment) {
        System.err.println("Error al aceptar: " + exc.getMessage());
    }
});
```

La primera linea dentro de `completed()` es `server.accept(null, this)`. Esto encadena la siguiente aceptacion antes incluso de procesar al cliente actual, garantizando que el servidor nunca deje de aceptar nuevas conexiones mientras haya capacidad en el grupo de hilos.

## Ejemplo completo: servidor echo asincrono

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.StandardCharsets;

public class ServidorEchoAsync {
    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open();
        server.bind(new InetSocketAddress(9000));
        System.out.println("Servidor async escuchando en puerto 9000");

        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel client, Void attach) {
                server.accept(null, this); // siguiente conexion
                ByteBuffer buf = ByteBuffer.allocate(4096);
                client.read(buf, buf, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer bytes, ByteBuffer buffer) {
                        if (bytes == -1) {
                            try { client.close(); } catch (Exception e) {}
                            return;
                        }
                        buffer.flip();
                        // Enviar eco
                        client.write(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                            @Override
                            public void completed(Integer written, ByteBuffer b) {
                                if (b.hasRemaining()) {
                                    client.write(b, b, this);
                                    return;
                                }
                                b.clear();
                                // Siguiente lectura
                                client.read(b, b, this);
                            }
                            @Override
                            public void failed(Throwable ex, ByteBuffer b) {
                                System.err.println("Error write: " + ex.getMessage());
                            }
                        });
                    }
                    @Override
                    public void failed(Throwable ex, ByteBuffer buffer) {
                        System.err.println("Error read: " + ex.getMessage());
                    }
                });
            }
            @Override
            public void failed(Throwable ex, Void attach) {
                System.err.println("Error accept: " + ex.getMessage());
            }
        });

        // Mantener la JVM viva
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

## Patron Proactor vs Reactor

El modelo asincrono de NIO.2 implementa el patron **Proactor**, mientras que el NIO clasico (Selector) sigue el patron **Reactor**. La diferencia fundamental radica en quien inicia la operacion y quien notifica su finalizacion.

| Aspecto | Reactor (NIO Selector) | Proactor (NIO.2 Async) |
|---------|----------------------|------------------------|
| Iniciador | Aplicacion (sondea) | Sistema operativo |
| Notificacion | "Listo para operar" | "Operacion completada" |
| E/S real | Aplicacion lee/escribe | Sistema lee/escribe, luego notifica |
| Hilos | 1 selector + N workers | Pool de hilos del grupo |
| Complejidad | Media (gestionar estados) | Alta (callbacks anidados) |

En Reactor, el selector notifica cuando un canal esta listo para lectura/escritura, pero la aplicacion debe ejecutar la operacion de E/S. En Proactor, la aplicacion solicita la E/S y el sistema operativo la ejecuta completamente, invocando el callback cuando los datos ya estan en el buffer. Windows con I/O Completion Ports es el ejemplo canonico de Proactor nativo; en Linux, la JVM emula Proactor sobre select/epoll.

## Accept con Future

Alternativamente, `accept()` devuelve un `Future<AsynchronousSocketChannel>`, permitiendo un estilo de programacion mas secuencial cuando se combina con `CompletableFuture`.

```java
AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open();
server.bind(new InetSocketAddress(9000));

Future<AsynchronousSocketChannel> futureClient = server.accept();
AsynchronousSocketChannel client = futureClient.get(); // bloquea

ByteBuffer buf = ByteBuffer.allocate(4096);
Future<Integer> futureRead = client.read(buf);
int bytes = futureRead.get();
```

Esta variante es menos practica para servidores de altas prestaciones, ya que `get()` bloquea el hilo, pero puede ser util en escenarios donde se sabe que solo llegara una conexion.

> El grupo de canales asincronos utiliza hilos daemon. Si el hilo principal termina, la JVM finaliza sin esperar a que las operaciones pendientes se completen. En servidores reales, el hilo principal debe permanecer activo mediante `Thread.sleep(Long.MAX_VALUE)`, un `CountDownLatch` o un bucle infinito controlado.
