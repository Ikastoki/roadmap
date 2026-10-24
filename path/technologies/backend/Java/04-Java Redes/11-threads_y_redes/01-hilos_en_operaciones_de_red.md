# Hilos en operaciones de red

La gestion de la concurrencia es el factor determinante en el rendimiento de las aplicaciones de red. Cada modelo de E/S en Java -- desde el clasico hilo por conexion hasta los virtual threads introducidos en Java 21 -- representa un equilibrio distinto entre simplicidad de programacion, consumo de recursos y capacidad de escalado. Comprender cuando aplicar cada modelo es esencial para disenar servidores que aprovechen eficientemente el hardware disponible.

## Thread-per-connection

El modelo mas intuitivo asigna un hilo del sistema operativo a cada conexion entrante. Cada hilo ejecuta un bucle de lectura/procesamiento/escritura de forma bloqueante.

```java
ServerSocket server = new ServerSocket(8080);
while (true) {
    Socket client = server.accept();
    new Thread(() -> {
        try {
            BufferedReader in = new BufferedReader(
                new InputStreamReader(client.getInputStream()));
            PrintWriter out = new PrintWriter(client.getOutputStream(), true);
            String line;
            while ((line = in.readLine()) != null) {
                out.println("Eco: " + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }).start();
}
```

**Problema:** Cada hilo del SO consume aproximadamente 1 MB de memoria de pila (stack). Con 1000 conexiones simultaneas, solo el stack ocupa ~1 GB. Ademas, el cambio de contexto (context switching) entre miles de hilos degrada el rendimiento del CPU de forma quadratica. Este modelo no escala mas alla de unos pocos cientos de conexiones.

## Thread pool con ExecutorService

En lugar de crear un hilo por conexion, se utiliza un pool de hilos reutilizables. Las tareas de atencion a clientes se envian como `Runnable` a un `ExecutorService`.

```java
ExecutorService pool = Executors.newFixedThreadPool(200);
ServerSocket server = new ServerSocket(8080);

while (true) {
    Socket client = server.accept();
    pool.execute(() -> {
        try {
            atenderCliente(client);
        } catch (IOException e) {
            e.printStackTrace();
        }
    });
}
```

El pool limita el numero de hilos activos simultaneamente. Las conexiones que exceden el tamanio del pool se encolan y esperan hasta que un hilo se libera. Esto evita la saturacion del sistema, pero introduce un problema de _head-of-line blocking_: si las 200 tareas estan bloqueadas en operaciones de red lentas (lecturas de clientes de baja latencia), el resto de conexiones esperan indefinidamente.

```java
// Estrategias de pool
Executors.newFixedThreadPool(200);     // tamanio fijo
Executors.newCachedThreadPool();       // crea hilos bajo demanda, reusa los inactivos
Executors.newWorkStealingPool(10);     // work-stealing (ForkJoinPool)
```

`newCachedThreadPool()` es peligroso para servidores: si llegan muchas conexiones, crea hilos sin limite, pudiendo agotar la memoria del sistema. `newFixedThreadPool()` es la opcion mas predecible para servidores.

## NIO Selector: un hilo, multiples conexiones

El modelo de E/S no bloqueante con `Selector` permite que un unico hilo geste miles de conexiones. El selector notifica cuando un canal esta listo para lectura, escritura o aceptacion, eliminando la necesidad de hilos bloqueados en espera.

```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false);
server.bind(new InetSocketAddress(8080));
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // bloquea hasta que haya eventos
    for (SelectionKey key : selector.selectedKeys()) {
        selector.selectedKeys().clear();
        if (key.isAcceptable()) {
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(4096);
            client.read(buf);
            // procesar y posiblemente registrar OP_WRITE
        }
    }
}
```

Este modelo escala a decenas de miles de conexiones con un solo hilo. Sin embargo, la complejidad del codigo aumenta significativamente: el estado de cada conexion debe gestionarse manualmente (buffers parciales, maquinas de estado de protocolo, y gestion de claves del selector).

## Virtual Threads (Java 21+)

Los virtual threads desacoplan la unidad de concurrencia del hilo del sistema operativo. Cada virtual thread ocupa aproximadamente 1 KB de memoria en lugar de 1 MB, y puede ser suspendido y reanudado por la JVM sin intervencion del kernel.

```java
try (var server = new ServerSocket(8080)) {
    while (true) {
        Socket client = server.accept();
        Thread.ofVirtual().start(() -> {
            try {
                atenderCliente(client);
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
}
```

Con virtual threads, se recupera la simplicidad del modelo thread-per-connection pero con la escalabilidad de NIO. La JVM monta los virtual threads sobre un pool reducido de hilos del SO (carrier threads), y cuando un virtual thread ejecuta una operacion bloqueante de E/S, la JVM lo desmonta y monta otro virtual thread en su lugar.

```java
// Tambien con ExecutorService
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    while (true) {
        Socket client = server.accept();
        executor.submit(() -> atenderCliente(client));
    }
}
```

## Comparativa de modelos

| Modelo | Hilos SO por conexion | Max conexiones tipico | Complejidad codigo | Uso de memoria |
|--------|----------------------|----------------------|--------------------|----------------|
| Thread-per-connection | 1 | ~500 | Baja | Muy alta |
| Thread pool | 1 (limitado) | ~2000 | Baja | Alta |
| NIO Selector | 0 (1 hilo para todas) | ~50000+ | Alta | Baja |
| Virtual Threads | 0 (compartidos) | ~50000+ | Baja | Muy baja |

> La eleccion del modelo depende del perfil de la aplicacion. Para APIs REST con operaciones cortas, un thread pool tradicional es suficiente. Para servidores de chat, streaming o websockets con larga duracion de conexion, NIO o virtual threads son practicamente obligatorios. Los virtual threads no son una bala de plata: operaciones CPU-intensivas dentro de un virtual thread siguen ocupando un carrier thread, limitando el paralelismo real.
