# Timeout en tests

Los timeouts en tests protegen contra ejecuciones infinitas, bloqueos perpetuos o degradaciones de rendimiento que convierten una suite de tests en un proceso que nunca termina. JUnit 5 ofrece dos mecanismos principales: la anotacion `@Timeout` y la asercion `assertTimeout`.

## assertTimeout

`assertTimeout` ejecuta un bloque de codigo y falla si supera la duracion especificada. Es util para operaciones que tienen un limite de tiempo conocido.

```java
import static org.junit.jupiter.api.Assertions.assertTimeout;
import java.time.Duration;

@Test
void operacionCompletaDentroDeLimite() {
    assertTimeout(Duration.ofMillis(500), () -> {
        servicio.ejecutarOperacionLarga();
    });
}
```

La variante `assertTimeoutPreemptively` aborta el hilo de ejecucion si se excede el tiempo, en lugar de esperar a que termine.

```java
@Test
void operacionNoExcedeLimiteConPreemption() {
    assertTimeoutPreemptively(Duration.ofSeconds(2), () -> {
        servicio.ejecutarPotencialmenteInfinito();
    });
}
```

> `assertTimeoutPreemptively` aborta el hilo, lo que puede dejar recursos sin liberar o estados inconsistentes si el codigo bajo prueba no es seguro para interrupcion. Se recomienda usarlo solo cuando sea estrictamente necesario.

## @Timeout

La anotacion `@Timeout` establece un limite temporal para la ejecucion de un metodo de test, de toda la clase o de todos los tests de una clase. Puede configurarse a nivel de metodo o de clase.

```java
import org.junit.jupiter.api.Timeout;
import java.util.concurrent.TimeUnit;

@Timeout(value = 5, unit = TimeUnit.SECONDS)
class ServicioLentoTest {

    @Test
    @Timeout(value = 500, unit = TimeUnit.MILLISECONDS)
    void operacionNormal() {
        servicio.ejecutar();
    }

    @Test
    void operacionLentaPeroDentroDelLimite() {
        servicio.ejecutarLento();
    }
}
```

La anotacion a nivel de clase aplica a todos los tests de la clase. La anotacion a nivel de metodo sobrescribe el valor de clase.

## SEPARATE_THREAD

Por defecto, `@Timeout` ejecuta el test en el mismo hilo y simplemente falla si se supera el tiempo. La configuracion `ThreadMode.SEPARATE_THREAD` ejecuta el test en un hilo separado y lo interrumpe al excederse.

```java
@Test
@Timeout(value = 1, threadMode = Timeout.ThreadMode.SEPARATE_THREAD)
void operacionEnHiloSeparado() {
    servicio.ejecutarPotencialmenteInfinito();
}
```

El modo `SEPARATE_THREAD` tiene las mismas consideraciones que `assertTimeoutPreemptively`: la interrupcion abrupta puede dejar el sistema en un estado inconsistente.

## Timeout como supuesto

A veces el timeout no debe fallar el test, sino indicar que el test no puede continuar y debe ser ignorado. JUnit 5 no tiene un mecanismo directo para esto, pero se puede simular con `assumeTrue` combinado con una medicion previa.

```java
@Test
void operacionSoloSiEsRapida() {
    long inicio = System.nanoTime();
    servicio.ejecutarPreparacion();
    long duracion = System.nanoTime() - inicio;

    assumeTrue(duracion < 1_000_000_000,
        "La preparacion fue demasiado lenta, se omite el test");

    servicio.ejecutarOperacionPrincipal();
}
```

## Cuando usar timeout

Los timeouts deben reservarse para:

- Operaciones con riesgo de bloqueo infinito (conexiones de red, locks, esperas condicionales).
- Algoritmos que deben cumplir un SLA en tiempo real.
- Tests que interactuan con sistemas externos no controlados.

En tests unitarios puros, sin I/O ni concurrencia, los timeouts rara vez son necesarios. Anadirlos indiscriminadamente incrementa el tiempo de ejecucion y enmascara problemas reales.
