# ServerSocketChannel

`ServerSocketChannel` es el canal para aceptar conexiones TCP entrantes dentro de `java.nio.channels`. Opera en conjuncion con `SocketChannel` y puede configurarse en modo no bloqueante para integrarse con el modelo `Selector`, permitiendo que un unico hilo geste miles de conexiones entrantes sin la penalizacion de crear un hilo por cada `accept()`.

## Apertura y configuracion

Se crea mediante `open()`, se configura como no bloqueante (opcional, pero es la razon de ser de NIO), y se asocia a una direccion local mediante `bind()`.

```java
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false); // modo no bloqueante
server.bind(new InetSocketAddress("0.0.0.0", 8080));
```

Sin `configureBlocking(false)`, el servidor se comporta como el clasico `ServerSocket` bloqueante. `bind()` acepta un `SocketAddress`; si se pasa `new InetSocketAddress(port)` sin especificar IP, escucha en todas las interfaces disponibles.

## Aceptar conexiones

En modo bloqueante, `accept()` se comporta como el `ServerSocket` tradicional, suspendiendo el hilo hasta que llegue una conexion. En modo no bloqueante, `accept()` retorna inmediatamente: `null` si no hay conexiones pendientes, o un `SocketChannel` si hay una.

```java
// Modo no bloqueante - bucle tipico
while (true) {
    SocketChannel client = server.accept();
    if (client != null) {
        client.configureBlocking(false);
        System.out.println("Cliente conectado: " + client.getRemoteAddress());
        // registrar client en Selector para lectura/escritura
    }
    // continuar con otras tareas...
}
```

Cada `SocketChannel` devuelto debe configurarse como no bloqueante de forma independiente, ya que hereda la configuracion del `ServerSocketChannel` en el momento de la aceptacion.

## Registro con Selector

El verdadero potencial de `ServerSocketChannel` emerge al combinarlo con `Selector`. El canal se registra para la operacion `OP_ACCEPT` y el selector notifica cuando hay conexiones entrantes, eliminando la necesidad del bucle de sondeo.

```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false);
server.bind(new InetSocketAddress(8080));
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // bloquea hasta que haya eventos
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> it = keys.iterator();

    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();

        if (key.isAcceptable()) {
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
            System.out.println("Cliente aceptado");
        } else if (key.isReadable()) {
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(4096);
            int bytes = client.read(buf);
            if (bytes == -1) {
                client.close();
                continue;
            }
            buf.flip();
            // procesar datos...
        }
    }
}
```

## Ejemplo completo de servidor NIO

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.Set;

public class ServidorNIO {
    public static void main(String[] args) throws Exception {
        Selector selector = Selector.open();
        ServerSocketChannel server = ServerSocketChannel.open();
        server.configureBlocking(false);
        server.bind(new InetSocketAddress(9000));
        server.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("Servidor NIO escuchando en puerto 9000");

        ByteBuffer buffer = ByteBuffer.allocate(4096);

        while (true) {
            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> it = keys.iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();

                if (!key.isValid()) continue;

                if (key.isAcceptable()) {
                    ServerSocketChannel srv = (ServerSocketChannel) key.channel();
                    SocketChannel client = srv.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                    System.out.println("Conectado: " + client.getRemoteAddress());
                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    buffer.clear();
                    int bytesRead = client.read(buffer);
                    if (bytesRead == -1) {
                        System.out.println("Desconectado: " + client.getRemoteAddress());
                        client.close();
                        continue;
                    }
                    buffer.flip();
                    String msg = StandardCharsets.UTF_8.decode(buffer).toString().trim();
                    System.out.println("Recibido: " + msg);
                    // Eco
                    buffer.flip();
                    client.write(buffer);
                }
            }
        }
    }
}
```

> Un error comun es no llamar a `it.remove()` despues de procesar cada `SelectionKey`. Si se omite, el selector seguira devolviendo la misma clave en iteraciones sucesivas, provocando un bucle infinito. Ademas, `selector.selectedKeys()` no se limpia automaticamente; el programador debe hacerlo.

## Metodos adicionales

```java
// Configurar opciones del socket del servidor
server.setOption(StandardSocketOptions.SO_RCVBUF, 65536);
server.setOption(StandardSocketOptions.SO_REUSEADDR, true);

// Obtener el ServerSocket tradicional subyacente (si es necesario)
ServerSocket legacySocket = server.socket();
```

`SO_REUSEADDR` debe activarse en servidores que necesitan reiniciarse rapidamente, ya que evita el error "address already in use" durante el periodo de espera del sistema operativo.
