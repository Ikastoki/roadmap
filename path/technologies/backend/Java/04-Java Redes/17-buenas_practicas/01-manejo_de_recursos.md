# Manejo de recursos

Los recursos de red (sockets, canales, streams, conexiones HTTP) son recursos limitados del sistema operativo. Si no se liberan correctamente, se agotan los descriptores de archivo del proceso, causando la conocida excepcion `Too many open files` o dejando sockets en estado `TIME_WAIT` que impiden reutilizar puertos. Java proporciona mecanismos para garantizar la liberacion incluso ante excepciones.

## Try-with-resources

Desde Java 7, el bloque try-with-resources cierra automaticamente cualquier recurso que implemente `AutoCloseable`. Es la forma preferida y mas segura de gestionar recursos de red.

```java
// Sin try-with-resources (propenso a fugas)
Socket socket = null;
try {
    socket = new Socket("ejemplo.com", 80);
    // usar socket...
} catch (IOException e) {
    log.error("Error", e);
} finally {
    if (socket != null) {
        try { socket.close(); } catch (IOException ignored) {}
    }
}

// Con try-with-resources (cierres automaticos)
try (Socket socket = new Socket("ejemplo.com", 80)) {
    // usar socket...
} catch (IOException e) {
    log.error("Error", e);
}
// socket.close() se llama automaticamente, incluso si hay excepcion
```

Multiples recursos se declaran separados por punto y coma; se cierran en orden inverso al de declaracion:

```java
try (Socket socket = new Socket("ejemplo.com", 80);
     InputStream in = socket.getInputStream();
     OutputStream out = socket.getOutputStream()) {
    // operaciones...
}
```

> `Socket`, `ServerSocket`, `InputStream`, `OutputStream`, `Reader`, `Writer`, `Channel`, `Selector` y `HttpClient` (Java 21+) implementan `AutoCloseable`. Siempre que sea posible, deben usarse dentro de try-with-resources.

## Cierre correcto de sockets

Cerrar un socket no es tan trivial como llamar a `close()`. Para un cierre ordenado en TCP se recomienda:

```java
public void cerrarSocket(Socket socket) {
    if (socket == null || socket.isClosed()) return;

    try {
        // Cierre ordenado: enviar FIN
        socket.shutdownOutput();
        // Leer hasta EOF (el remoto cierra su lado)
        while (socket.getInputStream().read() != -1) {}
    } catch (IOException e) {
        // El remoto ya cerro o hubo timeout
        log.debug("Cierre brusco del socket", e);
    } finally {
        try {
            socket.close();
        } catch (IOException e) {
            log.error("Error al cerrar socket", e);
        }
    }
}
```

`shutdownOutput()` envia una notificacion FIN al extremo remoto indicando que no se enviaran mas datos. El bucle de lectura consume hasta EOF, permitiendo al otro lado detectar el cierre ordenadamente.

## Canales NIO

Los canales de NIO (`SocketChannel`, `ServerSocketChannel`) tambien deben cerrarse correctamente:

```java
try (SocketChannel canal = SocketChannel.open()) {
    canal.connect(new InetSocketAddress("ejemplo.com", 80));
    ByteBuffer buf = ByteBuffer.allocate(1024);
    canal.read(buf);
} catch (IOException e) {
    log.error("Error en canal", e);
}
```

## Pool de conexiones

Crear una conexion TCP para cada operacion es ineficiente. El handshake de tres vias anade latencia a cada peticion. Un pool de conexiones reutiliza sockets establecidos, reduciendo la latencia y la carga del servidor.

```java
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.Semaphore;

public class PoolConexiones implements AutoCloseable {
    private final ConcurrentLinkedQueue<Socket> pool
            = new ConcurrentLinkedQueue<>();
    private final Semaphore permisos;

    public PoolConexiones(String host, int puerto, int max)
            throws IOException {
        this.permisos = new Semaphore(max);
        for (int i = 0; i < max; i++) {
            pool.add(new Socket(host, puerto));
        }
    }

    public Socket obtener() throws InterruptedException {
        permisos.acquire();
        return pool.poll();
    }

    public void devolver(Socket socket) {
        if (socket != null && !socket.isClosed()) {
            pool.offer(socket);
            permisos.release();
        }
    }

    @Override
    public void close() {
        pool.forEach(s -> {
            try { s.close(); } catch (IOException ignored) {}
        });
    }
}
```

> En produccion no se recomienda implementar pools manualmente. Apache Commons Pool 2, HikariCP (para JDBC) y los pools integrados en HttpClient (Java 11+) son opciones probadas y optimizadas.

## Timeouts

Un socket sin timeout puede bloquear indefinidamente. Siempre deben configurarse timeouts explicitos:

```java
Socket socket = new Socket();
socket.connect(new InetSocketAddress("ejemplo.com", 80), 5000); // connect timeout
socket.setSoTimeout(10000); // read timeout

// HttpClient (Java 11+)
HttpClient cliente = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(5))
        .build();

HttpRequest peticion = HttpRequest.newBuilder()
        .uri(URI.create("https://api.ejemplo.com"))
        .timeout(Duration.ofSeconds(10))
        .GET()
        .build();
```

- **connect timeout**: tiempo maximo para establecer la conexion TCP.
- **read timeout** (`setSoTimeout`): tiempo maximo entre llegada de datos (no tiempo total de operacion).
- **request timeout** (`HttpRequest.timeout`): tiempo maximo total de la peticion.

## Regla de oro

> Todo recurso que se abra debe cerrarse en el mismo bloque try-with-resources o en un finally. No delegar el cierre al garbage collector: `finalize()` esta deprecado desde Java 9 y no hay garantia de cuando se ejecuta.
