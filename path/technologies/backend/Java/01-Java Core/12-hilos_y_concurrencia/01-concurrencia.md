# Concurrencia

La concurrencia es la capacidad de un sistema para manejar multiples tareas en periodos de tiempo solapados. No implica ejecucion simultanea (eso es paralelismo), sino la ilusion de simultaneidad mediante la multiplexacion de recursos. En Java, la concurrencia esta soportada desde el nivel del lenguaje a traves del modelo de hilos, con el soporte de la maquina virtual que mapea hilos Java a hilos del sistema operativo.

## Concurrencia vs paralelismo

Concurrencia y paralelismo suelen confundirse. La concurrencia es sobre la estructura del programa: multiples tareas progresan independientemente, compartiendo el mismo recurso (CPU). El paralelismo es sobre la ejecucion: multiples tareas se ejecutan literalmente al mismo tiempo, requiriendo multiples nucleos o procesadores.

```java
// Concurrente: un solo nucleo multiplexa los hilos
// Paralelo: dos nucleos ejecutan ambos hilos simultaneamente
```

Un programa concurrente puede ejecutarse en un unico nucleo (alternando entre tareas). Un programa paralelo requiere hardware que lo soporte. La concurrencia es una propiedad del diseño; el paralelismo es una propiedad de la ejecucion.

## Race conditions

Una race condition ocurre cuando el resultado de una computacion depende del orden no determinista en que se ejecutan los hilos. El ejemplo clasico es la operacion de incremento:

```java
public class Contador {
    private int valor = 0;

    public void incrementar() {
        valor++; // NO es atomico: lectura, incremento, escritura
    }

    public int getValor() { return valor; }
}
```

Con dos hilos ejecutando `incrementar()` simultaneamente, pueden ocurrir secuencias como:

```
Hilo 1: lee valor=0
Hilo 2: lee valor=0
Hilo 1: escribe valor=1
Hilo 2: escribe valor=1  // Se perdio un incremento!
```

El resultado esperado era 2, pero se obtiene 1. La solucion es sincronizar el acceso al estado compartido.

## synchronized

`synchronized` garantiza exclusion mutua: solo un hilo puede ejecutar el bloque sincronizado a la vez, usando un monitor asociado a un objeto.

```java
public class ContadorSeguro {
    private int valor = 0;

    public synchronized void incrementar() {
        valor++;
    }

    // Alternativa: bloque sincronizado sobre this
    public void incrementarBloque() {
        synchronized (this) {
            valor++;
        }
    }
}
```

En metodos estaticos, el monitor es el objeto `Class` de la clase:

```java
public static synchronized void metodoEstatico() {
    // monitor sobre Clase.class
}
```

`synchronized` tambien garantiza visibilidad: los cambios realizados dentro del bloque son visibles para otros hilos que adquieran el mismo monitor posteriormente.

> Usar `synchronized` excesivamente degrada el rendimiento por contención de cerrojos. El objetivo es sincronizar la minima cantidad de codigo necesaria, no metodos enteros cuando solo unas lineas lo requieren.

## volatile

La palabra clave `volatile` garantiza que las lecturas y escrituras de una variable se realicen directamente sobre la memoria principal, no sobre la cache local del hilo. Esto soluciona problemas de visibilidad sin exclusion mutua.

```java
public class Bandera {
    private volatile boolean activo = true;

    public void detener() {
        activo = false; // visible inmediatamente para otros hilos
    }

    public void ejecutar() {
        while (activo) {
            // trabajar...
        }
    }
}
```

Sin `volatile`, el hilo que ejecuta `ejecutar()` podria cachear `activo=true` y nunca ver el cambio. `volatile` no proporciona atomicidad: `volatile int contador++` sigue siendo inseguro porque la operacion no es atomica.

## Locks: ReentrantLock

`ReentrantLock` ofrece funcionalidades mas flexibles que `synchronized`: intento de adquisicion con timeout, cerrojo interruptible, y multiples condiciones.

```java
import java.util.concurrent.locks.*;

public class CuentaBancaria {
    private final Lock lock = new ReentrantLock();
    private double saldo;

    public void depositar(double cantidad) {
        lock.lock();
        try {
            saldo += cantidad;
        } finally {
            lock.unlock(); // siempre liberar en finally
        }
    }

    public boolean intentarRetirar(double cantidad, long timeout, TimeUnit unit)
            throws InterruptedException {
        if (lock.tryLock(timeout, unit)) {
            try {
                if (saldo >= cantidad) {
                    saldo -= cantidad;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }
        return false; // no se pudo adquirir el lock en el tiempo dado
    }
}
```

El uso de `tryLock` con timeout evita bloqueos eternos. `ReentrantLock` es "reentrante": el mismo hilo puede adquirir el lock multiples veces sin bloquearse.

## ExecutorService y ThreadPool

Crear hilos explicitamente con `new Thread()` es ineficiente y propenso a errores. `ExecutorService` abstrae la ejecucion de tareas en un pool de hilos reutilizables.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
// Alternativas:
// Executors.newCachedThreadPool()  - pool dinamico
// Executors.newSingleThreadExecutor() - un solo hilo
// Executors.newScheduledThreadPool() - ejecucion programada

executor.submit(() -> System.out.println("Tarea ejecutada"));

// Apagado ordenado
executor.shutdown();

// Esperar a que terminen las tareas pendientes
executor.awaitTermination(5, TimeUnit.SECONDS);
```

El `ThreadPool` reutiliza hilos activos, evita el overhead de creacion/destruccion, y limita el numero de hilos concurrentes para evitar saturacion del sistema.

## Callable y Future

`Callable` es similar a `Runnable` pero puede devolver un resultado y lanzar excepciones verificadas. `Future` representa el resultado pendiente de una tarea asincrona.

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

Callable<Integer> tarea = () -> {
    Thread.sleep(1000);
    return 42;
};

Future<Integer> future = executor.submit(tarea);

// Hacer otras cosas mientras se ejecuta la tarea...

Integer resultado = future.get(); // bloquea hasta obtener el resultado
```

`Future` soporta cancelacion, comprobacion de completitud, y timeout en la obtencion:

```java
Integer resultado = future.get(2, TimeUnit.SECONDS); // timeout
boolean cancelado = future.cancel(true);
boolean terminado = future.isDone();
```

## CompletableFuture

`CompletableFuture` extiende `Future` con capacidades de programacion funcional y composicion asincrona. Permite encadenar llamadas, manejar errores, y combinar multiples futuros sin bloquear.

```java
CompletableFuture.supplyAsync(() -> {
    // Tarea asincrona en ForkJoinPool.commonPool()
    return "resultado";
}).thenApply(s -> s.toUpperCase())
  .thenAccept(System.out::println)
  .exceptionally(ex -> {
      System.err.println("Error: " + ex.getMessage());
      return null;
  });
```

Composicion de multiples futuros:

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hola");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "Mundo");

CompletableFuture<String> combinado = f1.thenCombine(f2, (a, b) -> a + " " + b);

// Esperar todos
CompletableFuture<Void> todos = CompletableFuture.allOf(f1, f2);

// Ejecutar accion cuando cualquiera termine
CompletableFuture<Object> cualquiera = CompletableFuture.anyOf(f1, f2);
```

> A diferencia de `Future.get()`, que bloquea el hilo actual, los metodos `then*` de `CompletableFuture` registran callbacks que se ejecutan cuando el resultado esta disponible, sin bloquear. Esto lo convierte en la opcion preferida para programacion asincrona no bloqueante en Java.
