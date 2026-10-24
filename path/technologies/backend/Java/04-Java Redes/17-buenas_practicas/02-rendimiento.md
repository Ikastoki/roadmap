# Rendimiento

El rendimiento de una aplicacion de red no depende solo de la latencia del medio fisico o del ancho de banda disponible. La configuracion del lado de la aplicacion, el tamano de los buffers, la eleccion entre I/O bloqueante y no bloqueante, y la reutilizacion de conexiones tienen un impacto directo en la capacidad de procesamiento y en la latencia percibida por el usuario.

## Buffer size optimo

El tamano del buffer de lectura/escritura influye en el numero de llamadas al sistema y en la eficiencia del protocolo TCP.

```java
Socket socket = new Socket();
socket.setReceiveBufferSize(65536); // 64 KB
socket.setSendBufferSize(65536);

// O configurar antes de conectar
SocketChannel channel = SocketChannel.open();
channel.setOption(StandardSocketOptions.SO_RCVBUF, 65536);
channel.setOption(StandardSocketOptions.SO_SNDBUF, 65536);
```

- **8 KB**: valor por defecto en muchos sistemas. Suficiente para aplicaciones con trafico ligero.
- **16-32 KB**: recomendado para aplicaciones web y APIs REST.
- **64 KB**: optimo para transferencia de archivos grandes o streaming.

> TCP utiliza el doble del tamano configurado para la ventana de recepcion (receive window). Un buffer de 64 KB permite una ventana de 128 KB, lo que mejora el rendimiento en enlaces con alta latencia (bandwidth-delay product).

Para lecturas, usar buffers del tamano del buffer del socket:

```java
byte[] buffer = new byte[8192]; // 8 KB
int leidos;
while ((leidos = in.read(buffer)) != -1) {
    out.write(buffer, 0, leidos);
}
```

## Non-blocking I/O

El I/O bloqueante asigna un hilo por conexion. Con miles de conexiones simultaneas, el cambio de contexto (context switching) y el consumo de memoria por pila de hilo (tipicamente 1 MB por hilo) degradan el rendimiento.

NIO (New I/O) con `Selector` permite que un unico hilo gestione multiples canales:

```java
import java.nio.channels.*;
import java.util.Iterator;

public class ServidorNIO {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.bind(new InetSocketAddress(8080));
        ssc.configureBlocking(false);
        ssc.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            selector.select(); // bloquea hasta que hay eventos
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();

                if (key.isAcceptable()) {
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);
                    sc.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    SocketChannel sc = (SocketChannel) key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(8192);
                    sc.read(buf);
                    // procesar datos...
                }
            }
        }
    }
}
```

Un selector es eficiente porque el sistema operativo notifica cuando hay datos disponibles, evitando polling. Para aplicaciones HTTP, frameworks como Netty, Vert.x o Spring WebFlux implementan este modelo.

> NIO no es siempre mas rapido que I/O bloqueante. Para aplicaciones con pocas conexiones simultaneas (decenas), el modelo bloqueante es mas simple y tiene menor overhead. NIO brilla con miles de conexiones concurrentes.

## Connection pooling

Establecer una conexion TCP implica tres mensajes (SYN, SYN-ACK, ACK) mas el handshake TLS si aplica. Reutilizar conexiones elimina este coste.

```java
HttpClient cliente = HttpClient.newBuilder()
        .executor(Executors.newFixedThreadPool(10))
        .build();

// HttpClient (Java 11+) mantiene un pool interno de conexiones
// y las reutiliza automaticamente cuando es posible
```

Para APIs REST, el pool debe estar dimensionado segun el numero de peticiones simultaneas esperadas:

```java
PoolingHttpClientConnectionManager pool =
        new PoolingHttpClientConnectionManager();
pool.setMaxTotal(200);              // conexiones totales
pool.setDefaultMaxPerRoute(50);     // por host (route)

CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(pool)
        .build();
```

## Keep-Alive

HTTP keep-alive (persistent connection) reutiliza la misma conexion TCP para multiples peticiones HTTP, evitando el coste de abrir y cerrar sockets repetidamente. HTTP/1.1 lo habilita por defecto; HTTP/2 lo presupone.

```java
// Con HttpURLConnection (legacy)
HttpURLConnection con = (HttpURLConnection)
        new URL("https://api.ejemplo.com").openConnection();
con.setRequestProperty("Connection", "keep-alive");
// En HTTP/1.1 es implicito
```

En el servidor, configurar el timeout de keep-alive para no mantener conexiones inactivas indefinidamente:

```java
// En Tomcat (server.xml)
<Connector port="8080"
           keepAliveTimeout="30000"
           maxKeepAliveRequests="100"/>
```

## Otras optimizaciones

- **TCP_NODELAY**: desactiva el algoritmo de Nagle, que acumula paquetes pequenos antes de enviarlos. Para APIs interactivas, activarlo reduce latencia.

```java
socket.setTcpNoDelay(true);
```

- **SO_REUSEADDR**: permite reutilizar un puerto en estado `TIME_WAIT` inmediatamente.

```java
serverSocket.setReuseAddress(true);
```

- **Compresion**: comprimir cuerpos HTTP con gzip reduce el ancho de banda a costa de CPU.

```java
HttpRequest peticion = HttpRequest.newBuilder()
        .header("Accept-Encoding", "gzip")
        .build();
```

> Medir antes de optimizar. Usar herramientas de profiling (JProfiler, VisualVM) y monitorizacion (Prometheus, Micrometer) para identificar cuellos de botella reales. Optimizar sin datos es conjetura.
