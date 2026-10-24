# Virtual Threads (Java 21+)

Los virtual threads, introducidos de forma definitiva en Java 21 (JEP 444), representan el cambio mas significativo en el modelo de concurrencia de Java desde sus origenes. Un virtual thread es un hilo gestionado enteramente por la JVM que se ejecuta sobre un pool de hilos del sistema operativo (carrier threads), permitiendo crear millones de hilos con un consumo minimo de memoria. Para aplicaciones de red con E/S bloqueante, los virtual threads ofrecen la simplicidad del modelo thread-per-connection con la escalabilidad del NIO clasico.

## Creacion de virtual threads

Java 21 proporciona varias formas de crear virtual threads, todas en `Thread.Builder` y sus metodos de factoria.

```java
// Metodo 1: Thread.ofVirtual()
Thread vThread = Thread.ofVirtual()
    .name("worker-1")
    .start(() -> {
        System.out.println("Ejecutando en virtual thread: " + Thread.currentThread());
    });

// Metodo 2: Thread.startVirtualThread()
Thread vThread2 = Thread.startVirtualThread(() -> {
    System.out.println("Virtual thread iniciado directamente");
});

// Metodo 3: Executors.newVirtualThreadPerTaskExecutor()
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> procesar());
    executor.submit(() -> procesar());
}
```

La JVM no garantiza el orden de ejecucion ni en que carrier thread se ejecutara cada virtual thread. Lo unico seguro es que, desde la perspectiva del codigo, la semantica es identica a la de un thread plataforma.

## Ventajas frente a threads del SO

La diferencia fundamental radica en el coste por hilo.

| Aspecto | Thread plataforma | Virtual Thread |
|---------|------------------|----------------|
| Stack tipico | ~1 MB (fijo) | ~1 KB (dinamico) |
| Maximos por GB RAM | ~1000 | ~1.000.000+ |
| Creacion | Llamada al kernel | Operacion en memoria |
| Context switch | Kernel ~1-10 us | JVM ~0.1-1 us |
| Bloqueo de E/S | Bloquea carrier | Desmonta y monta otro |

Cuando un virtual thread ejecuta una operacion bloqueante de E/S (como `Socket.read()` o `InputStream.read()`), la JVM _desmonta_ el virtual thread del carrier thread y lo reemplaza por otro virtual thread listo para ejecutar. Cuando la operacion de E/S se completa, el virtual thread se _monta_ nuevamente en cualquier carrier thread disponible. Este mecanismo, llamado _continuation_, es transparente para el programador.

## Ideal para E/S bloqueante

Los virtual threads estan disenados especificamente para aplicaciones donde los hilos pasan la mayor parte del tiempo bloqueados en E/S. Un servidor HTTP tradicional que asigna un hilo por peticion ve limitada su escalabilidad por el numero de hilos del SO. Con virtual threads, ese mismo servidor puede manejar decenas de miles de peticiones simultaneas.

```java
// Servidor HTTP con virtual threads - simple y escalable
import java.io.*;
import java.net.*;

public class ServidorVirtual {
    public static void main(String[] args) throws Exception {
        try (var server = new ServerSocket(8080)) {
            System.out.println("Servidor virtual threads en puerto 8080");
            while (true) {
                Socket client = server.accept();
                Thread.ofVirtual().start(() -> {
                    try {
                        atender(client);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });
            }
        }
    }

    static void atender(Socket client) throws IOException {
        try (client;
             var in = new BufferedReader(
                 new InputStreamReader(client.getInputStream()));
             var out = new PrintWriter(client.getOutputStream(), true)) {

            String line;
            while ((line = in.readLine()) != null) {
                if (line.isEmpty()) break;
                out.println("Eco: " + line);
            }
        }
    }
}
```

Cada `Thread.ofVirtual().start()` crea un virtual thread que se suspende automaticamente cuando lee del socket y se reanuda cuando hay datos disponibles. La JVM gestiona todo el montaje/desmontaje sobre los carrier threads de forma transparente.

## Ejemplo con ExecutorService

```java
import java.util.concurrent.*;

public class ServidorConPool {
    public static void main(String[] args) throws Exception {
        var server = new ServerSocket(9090);
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            while (true) {
                Socket client = server.accept();
                executor.submit(() -> {
                    try {
                        manejarCliente(client);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });
            }
        }
    }

    static void manejarCliente(Socket socket) throws IOException {
        try (socket;
             var in = new DataInputStream(socket.getInputStream());
             var out = new DataOutputStream(socket.getOutputStream())) {

            byte[] buf = new byte[4096];
            int n;
            while ((n = in.read(buf)) != -1) {
                out.write(buf, 0, n);
            }
        }
    }
}
```

`newVirtualThreadPerTaskExecutor()` devuelve un `ExecutorService` que crea un nuevo virtual thread por cada tarea enviada, sin necesidad de gestionar pools ni tamanios maximos.

## Limitaciones y consideraciones

No todas las operaciones de E/S desmontan el virtual thread del carrier. Las operaciones que no son interceptables por la JVM (como E/S nativa via JNI, ciertas operaciones de archivos en algunos sistemas, o synchronized blocks profundamente anidados) _pinzan_ el virtual thread al carrier thread, impidiendo el desmontaje.

```java
// Esto PINZA el virtual thread al carrier:
synchronized (lock) {
    Thread.sleep(1000); // bloquea el carrier thread
}
```

Para evitar el pinning, debe evitarse el uso de `synchronized` dentro de operaciones bloqueantes largas en virtual threads. En su lugar, usar `ReentrantLock`:

```java
// Alternativa que permite el desmontaje:
private final Lock lock = new ReentrantLock();

lock.lock();
try {
    Thread.sleep(1000); // se desmonta correctamente
} finally {
    lock.unlock();
}
```

> Los virtual threads no estan disenados para tareas intensivas en CPU. Si el codigo realiza computacion pesada sin E/S, los virtual threads no ofrecen ventaja sobre los threads plataforma, ya que no hay oportunidad de desmontaje. En ese caso, el paralelismo real esta limitado por el numero de cores, y es mejor usar un pool de threads plataforma con `newFixedThreadPool(Runtime.getRuntime().availableProcessors())`.
