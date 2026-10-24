# Sockets con hilos

Un servidor TCP que atiende una sola conexion no es util en un entorno real. La concurrentia es inherente al modelo cliente-servidor: cada conexion debe ser atendida sin bloquear la aceptacion de nuevas conexiones. El modo mas directo de lograrlo en Java es asignar un hilo a cada conexion.

## Patron thread-per-connection

El patron consiste en que el bucle principal del servidor acepta conexiones y, por cada una, crea un nuevo hilo que se encarga exclusivamente de la comunicacion con ese cliente. El hilo principal vuelve inmediatamente a `accept()` para atender al siguiente cliente.

```java
public class ServidorThreadPerConnection {
    public static void main(String[] args) throws IOException {
        try (ServerSocket ss = new ServerSocket(9090)) {
            System.out.println("Servidor multihilo en puerto 9090");
            while (true) {
                Socket cliente = ss.accept();
                new Thread(new ManejadorCliente(cliente)).start();
            }
        }
    }
}

class ManejadorCliente implements Runnable {
    private final Socket socket;

    public ManejadorCliente(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        try (socket;
             BufferedReader in = new BufferedReader(
                 new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(
                 new OutputStreamWriter(socket.getOutputStream()), true)) {

            String linea;
            while ((linea = in.readLine()) != null) {
                out.println("Echo: " + linea);
            }
        } catch (IOException e) {
            System.err.println("Error con cliente: " + e.getMessage());
        }
    }
}
```

Cada `ManejadorCliente` ejecuta en su propio hilo. El `try-with-resources` sobre `socket` asegura el cierre automatico de la conexion cuando el bucle de lectura termina (el cliente cierra o hay un error).

## Pool de hilos con ExecutorService

Crear un hilo por conexion tiene coste: la creacion de un hilo nativo del sistema operativo implica reserva de pila (~1 MB por defecto en muchas JVM), llamadas al sistema y overhead de planificacion. Para mitigarlo, `ExecutorService` mantiene un pool de hilos reutilizables.

```java
public class ServidorConPool {
    public static void main(String[] args) throws IOException {
        ExecutorService pool = Executors.newFixedThreadPool(10);
        try (ServerSocket ss = new ServerSocket(9090)) {
            while (true) {
                Socket cliente = ss.accept();
                pool.execute(new ManejadorCliente(cliente));
            }
        }
    }
}
```

`newFixedThreadPool(10)` crea un pool con 10 hilos. Si los 10 estan ocupados, las nuevas tareas se encolan en una cola interna (un `LinkedBlockingQueue`) hasta que un hilo se libere. Esto evita la sobrecarga del sistema bajo picos de conexion.

Existen otras variantes:

- `newCachedThreadPool()`: crea hilos segun demanda y reutiliza los inactivos. Si un hilo lleva 60 segundos sin uso, lo elimina. Adecuado para cargas impredecibles con tareas ligeras.
- `newSingleThreadExecutor()`: un unico hilo ejecuta todas las tareas secuencialmente. Garantiza que no hay concurrencia.
- `newScheduledThreadPool(int)`: permite ejecutar tareas con retardo o periodicamente, util para envio de heartbeats o reintentos.

> El tamano optimo del pool depende de la naturaleza de la tarea. Para tareas bound a CPU se recomienda `Runtime.getRuntime().availableProcessors() + 1`. Para tareas bound a E/S, como la espera en `read()`, el pool puede ser mayor porque los hilos pasan la mayor parte del tiempo bloqueados.

## Limitaciones del modelo bloqueante

Aunque el pool de hilos mejora el comportamiento, el modelo thread-per-connection tiene limitaciones fundamentales que lo hacen inadecuado para cargas extremas.

### Context switching

Cada vez que la JVM planifica un hilo diferente, el sistema operativo debe guardar el estado del hilo actual (registros, contador de programa, puntero de pila) y restaurar el del hilo entrante. Con miles de hilos activos, el tiempo dedicado a context switching puede superar al tiempo dedicado a procesar datos reales.

### Overhead de memoria

Cada hilo nativo reserva una pila independiente. Con el valor por defecto de 1 MB, 10.000 conexiones supondrian 10 GB solo en pilas de hilos, antes de contar el heap de la aplicacion. Reducir el tamano de pila con `-Xss256k` ayuda, pero sigue siendo un limite severo.

### Inactividad costosa

En un servidor tipico, muchas conexiones permanecen inactivas la mayor parte del tiempo (por ejemplo, conexiones HTTP keep-alive esperando la siguiente peticion). Aun asi, cada hilo consume recursos del planificador, que debe iterar sobre todos los hilos para decidir cual ejecutar.

### Soluciones alternativas

NIO con selectors (un hilo gestiona miles de canales) y virtual threads (hilos ligeros que no corresponden 1:1 con hilos del SO) resuelven estas limitaciones. Con virtual threads, el patron thread-per-conexion recupera su simplicidad original sin los problemas de escalabilidad:

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    while (true) {
        Socket cliente = ss.accept();
        executor.submit(() -> new ManejadorCliente(cliente).run());
    }
}
```

Cada virtual thread pesa unos pocos kilobytes y se suspende automaticamente cuando bloquea en E/S, permitiendo que el carrier thread (hilo del SO subyacente) atienda a otros virtual threads.
