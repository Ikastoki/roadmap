# Hilos

Un hilo (thread) es la unidad minima de ejecucion que puede planificar el sistema operativo. En Java, la clase `Thread` y la interfaz `Runnable` proporcionan los mecanismos fundamentales para crear y controlar hilos. Cada hilo tiene su propia pila de llamadas y contador de programa, pero comparte el heap y los recursos del proceso con los demas hilos.

## Creacion de hilos

### Thread

La forma mas directa es extender `Thread` y sobrescribir `run()`:

```java
class MiHilo extends Thread {
    @Override
    public void run() {
        System.out.println("Ejecutando en: " + Thread.currentThread().getName());
    }
}

MiHilo hilo = new MiHilo();
hilo.start(); // inicia el hilo, no llamar a run() directamente
```

### Runnable

Prefiere implementar `Runnable` sobre extender `Thread`, ya que no consume la unica herencia de clase y separa la tarea del mecanismo de ejecucion:

```java
Runnable tarea = () -> {
    System.out.println("Ejecutando en: " + Thread.currentThread().getName());
};

Thread hilo = new Thread(tarea);
hilo.start();
```

Las lambdas hacen que implementar `Runnable` sea especialmente conciso.

### Callable

A diferencia de `Runnable`, `Callable` puede devolver un resultado y lanzar excepciones verificadas. Requiere un `ExecutorService` para su ejecucion:

```java
Callable<Integer> tarea = () -> {
    Thread.sleep(1000);
    return 42;
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(tarea);
Integer resultado = future.get(); // 42
```

## Ciclo de vida

Un hilo atraviesa varios estados definidos en `Thread.State`:

```
NEW -> RUNNABLE -> BLOCKED | WAITING | TIMED_WAITING -> TERMINATED
```

### NEW

El hilo ha sido creado pero no se ha invocado `start()`. Aun no esta vivo:

```java
Thread hilo = new Thread(tarea);
System.out.println(hilo.getState()); // NEW
```

### RUNNABLE

Despues de llamar a `start()`, el hilo esta en RUNNABLE: listo para ejecutar o ejecutandose. Depende del planificador del SO asignarle tiempo de CPU.

```java
hilo.start();
System.out.println(hilo.getState()); // RUNNABLE
```

### BLOCKED

El hilo esta bloqueado esperando adquirir un monitor (entrar en un bloque `synchronized`).

### WAITING

El hilo espera indefinidamente a que otro hilo lo notifique. Metodos como `wait()`, `join()` sin timeout, o `LockSupport.park()` producen este estado.

### TIMED_WAITING

Similar a WAITING pero con tiempo limite. `Thread.sleep()`, `wait(timeout)`, `join(timeout)`.

### TERMINATED

El metodo `run()` ha finalizado normalmente o por una excepcion no capturada.

```java
hilo.join(); // esperar a que termine
System.out.println(hilo.getState()); // TERMINATED
```

## Metodos de control de hilos

### start()

Inicia el hilo, lo pone en estado RUNNABLE. Nunca debe llamarse a `run()` directamente, ya que eso ejecutaria la tarea en el hilo actual sin crear uno nuevo.

### join()

Espera a que el hilo especificado termine. Permite sincronizacion entre hilos:

```java
Thread hilo = new Thread(() -> {
    // trabajo pesado
});
hilo.start();
hilo.join(); // el hilo actual espera a que hilo termine
System.out.println("El hilo ha terminado");
```

### sleep()

Pausa la ejecucion del hilo actual durante el tiempo especificado, sin liberar el monitor:

```java
try {
    Thread.sleep(2000); // 2 segundos
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restaurar estado de interrupcion
}
```

`sleep()` lanza `InterruptedException` si otro hilo interrumpe al hilo dormido. La practica recomendada es restaurar el estado de interrupcion.

### yield()

Sugiere al planificador que el hilo actual esta dispuesto a ceder la CPU. Es solo una sugerencia; el planificador puede ignorarla.

```java
Thread.yield(); // "Sugiero ceder la CPU"
```

### interrupt()

Solicita la interrupcion de un hilo. No lo detiene forzosamente; establece una bandera de interrupcion que el hilo debe comprobar:

```java
hilo.interrupt(); // establece el flag de interrupcion

// En el hilo target:
while (!Thread.currentThread().isInterrupted()) {
    // trabajar...
}

// O usando el flag explícitamente:
if (Thread.interrupted()) { // limpia el flag
    // limpiar y salir
}
```

> `interrupt()` no mata el hilo. La cooperacion es esencial: el hilo interrumpido decide como responder. Los metodos que lanzan `InterruptedException` (sleep, wait, join) limpian el flag al lanzar la excepcion.

## Daemon threads

Los hilos daemon son hilos de segundo plano que no impiden que la JVM termine. Cuando todos los hilos no-daemon finalizan, la JVM termina abruptamente los hilos daemon restantes.

```java
Thread hilo = new Thread(tarea);
hilo.setDaemon(true); // debe llamarse antes de start()
hilo.start();
```

Los hilos daemon son utiles para tareas de mantenimiento, monitoreo, o limpieza periodica. No deben usarse para tareas criticas o que requieran finalizacion ordenada.

## Thread safety

La seguridad en hilos (thread safety) es la propiedad que garantiza que una clase se comporta correctamente cuando es accedida por multiples hilos, independientemente de la planificacion del sistema operativo. Se consigue mediante:

- **Inmutabilidad**: objetos que nunca cambian despues de su construccion (String, Optional, records).
- **Sincronizacion**: exclusion mutua con `synchronized` o `Lock`.
- **Variables atomicas**: `AtomicInteger`, `AtomicReference`, etc.
- **Estructuras concurrentes**: `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`.
- **ThreadLocal**: variables confinadas al hilo actual.

```java
// ThreadLocal: cada hilo tiene su propia copia
private static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String formatear(Date fecha) {
    return dateFormat.get().format(fecha);
}
```

`SimpleDateFormat` no es thread-safe; `ThreadLocal` evita la contención al dar a cada hilo su propia instancia.

```java
// AtomicInteger: operaciones atomicas sin sincronizacion explicita
AtomicInteger contador = new AtomicInteger(0);

public void incrementar() {
    contador.incrementAndGet();
}
```

Las clases atomicas usan instrucciones CAS (Compare-And-Swap) del hardware, ofreciendo mayor rendimiento que `synchronized` en operaciones simples.
