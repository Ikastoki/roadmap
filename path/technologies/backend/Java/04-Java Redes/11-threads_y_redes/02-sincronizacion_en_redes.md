# Sincronizacion en redes

Cuando multiples hilos manejan conexiones de red simultaneamente, el acceso concurrente a recursos compartidos se convierte en la principal fuente de bugs y problemas de rendimiento. El estado de las conexiones, las colas de mensajes, las caches de sesiones y los contadores de estadisticas deben protegerse adecuadamente para evitar condiciones de carrera, corrupcion de datos y puntos muertos. Java proporciona un arsenal de mecanismos de sincronizacion que cubren desde la exclusion mutua clasica hasta estructuras de datos concurrentes sin bloqueo.

## Recursos compartidos tipicos en servidores de red

Antes de elegir el mecanismo de sincronizacion, hay que identificar que recursos se comparten entre hilos:

- **Mapa de sesiones**: asociacion entre ID de conexion y estado del cliente.
- **Colas de mensajes**: mensajes pendientes de enviar a cada cliente.
- **Cache de respuestas**: resultados de operaciones costosas cacheados en memoria.
- **Contadores**: metricas como peticiones atendidas, bytes transferidos, conexiones activas.
- **Pool de conexiones a bases de datos**: recursos compartidos entre peticiones.

## synchronized

El bloque `synchronized` es el mecanismo mas basico y seguro para exclusion mutua. Garantiza que solo un hilo ejecute la seccion critica a la vez.

```java
public class GestorSesiones {
    private final Map<String, Sesion> sesiones = new HashMap<>();

    public synchronized Sesion obtener(String id) {
        return sesiones.get(id);
    }

    public synchronized void agregar(String id, Sesion s) {
        sesiones.put(id, s);
    }
}
```

El problema de `synchronized` en servidores de red es que convierte todo el metodo en un punto de contencion. Si 200 hilos quieren obtener sesiones simultaneamente, solo uno accede a la vez, reduciendo el rendimiento. Ademas, `synchronized` no ofrece mecanismos de espera condicional mas alla de `wait()/notify()`.

## ReentrantLock

`ReentrantLock` proporciona funcionalidad equivalente a `synchronized` pero con caracteristicas adicionales: interrupcion, tiempo de espera, y sondeo de bloqueo.

```java
public class ColaMensajes {
    private final Queue<Mensaje> cola = new LinkedList<>();
    private final ReentrantLock lock = new ReentrantLock();

    public void encolar(Mensaje msg) {
        lock.lock();
        try {
            cola.offer(msg);
        } finally {
            lock.unlock();
        }
    }

    public Mensaje despachar() {
        lock.lock();
        try {
            return cola.poll();
        } finally {
            lock.unlock();
        }
    }
}
```

`tryLock()` permite intentar adquirir el lock sin bloquearse, util para evitar esperas indefinidas en operaciones de red donde el tiempo de respuesta es critico.

```java
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        // operacion critica
    } finally {
        lock.unlock();
    }
} else {
    // camino alternativo, log o reintento
}
```

## ReadWriteLock

Cuando el recurso compartido se lee con mucha mas frecuencia de la que se escribe (como una cache de resolucion DNS o un mapa de rutas), `ReadWriteLock` permite que multiples hilos lean concurrentemente mientras la escritura sigue siendo exclusiva.

```java
public class CacheRespuestas {
    private final Map<String, Respuesta> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public Respuesta obtener(String clave) {
        rwLock.readLock().lock();
        try {
            return cache.get(clave);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void almacenar(String clave, Respuesta resp) {
        rwLock.writeLock().lock();
        try {
            cache.put(clave, resp);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

En servidores de red donde las lecturas superan ampliamente a las escrituras (ratio 10:1 o mayor), `ReadWriteLock` puede duplicar el rendimiento comparado con `synchronized`.

## ConcurrentHashMap

Para mapas compartidos entre hilos, `ConcurrentHashMap` es casi siempre la mejor opcion. Internamente segmenta el mapa en varias regiones (buckets), permitiendo acceso concurrente a diferentes segmentos sin bloqueo global. Ademas, ofrece operaciones atomicas compuestas.

```java
ConcurrentHashMap<String, Sesion> sesiones = new ConcurrentHashMap<>();

// Operaciones atomicas
sesiones.putIfAbsent("user1", new Sesion());
sesiones.computeIfAbsent("user2", id -> crearSesion(id));

// Iteracion segura sin ConcurrentModificationException
for (Map.Entry<String, Sesion> e : sesiones.entrySet()) {
    System.out.println(e.getKey());
}
```

`compute()`, `merge()` y `computeIfAbsent()` realizan actualizaciones atomicas compuestas sin necesidad de sincronizacion externa, eliminando la ventana de condicion de carrera tipica del patron "get, modificar, put".

## BlockingQueue

Las colas bloqueantes son el mecanismo ideal para el patron productor-consumidor entre hilos que manejan conexiones de red. `ArrayBlockingQueue` (tamanio fijo) y `LinkedBlockingQueue` (tamanio opcional) proporcionan insercion y extraccion con bloqueo automatico.

```java
BlockingQueue<Mensaje> colaMensajes = new LinkedBlockingQueue<>(10000);

// Hilo productor (lectura de socket)
colaMensajes.put(mensaje); // bloquea si la cola esta llena

// Hilo consumidor (procesamiento)
Mensaje msg = colaMensajes.take(); // bloquea si la cola esta vacia
```

`offer(time, unit)` y `poll(time, unit)` permiten esperas con tiempo maximo, evitando bloqueos permanentes.

## AtomicInteger y clases atomicas

Para contadores y acumuladores simples, las clases del paquete `java.util.concurrent.atomic` proporcionan operaciones sin bloqueo (lock-free) mediante instrucciones CAS (Compare-And-Swap) del procesador.

```java
public class Metricas {
    private final AtomicInteger peticiones = new AtomicInteger(0);
    private final AtomicLong bytesTransferidos = new AtomicLong(0);
    private final AtomicInteger conexionesActivas = new AtomicInteger(0);

    public void registrarPeticion(int bytes) {
        peticiones.incrementAndGet();
        bytesTransferidos.addAndGet(bytes);
    }

    public void conectar() {
        conexionesActivas.incrementAndGet();
    }

    public void desconectar() {
        conexionesActivas.decrementAndGet();
    }
}
```

Las clases atomicas son significativamente mas rapidas que cualquier mecanismo con locks, ya que no implican cambio de contexto ni suspension del hilo. La contrapartida es que solo cubren operaciones sobre una unica variable.

> La sincronizacion debe elegirse segun el perfil de acceso. `ConcurrentHashMap` reemplaza practicamente siempre a `HashMap + synchronized`. Las clases atomicas son ideales para contadores. `ReadWriteLock` solo mejora el rendimiento cuando las lecturas son mucho mas frecuentes que las escrituras. En caso de duda, `ReentrantLock` ofrece la maxima flexibilidad. La regla de oro: usar la estructura concurrente mas simple que resuelva el problema sin contencion innecesaria.
