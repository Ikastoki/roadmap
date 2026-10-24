# JMH (Java Microbenchmark Harness)

JMH es una libreria desarrollada por el equipo de rendimiento de la JVM (OpenJDK) para escribir microbenchmarks en Java. Su proposito es medir el rendimiento de fragmentos de codigo con precision, eliminando los sesgos introducidos por la compilacion JIT, la recoleccion de basura, la optimizacion del bucle caliente y otros fenomenos que distorsionan las mediciones en la JVM.

Escribir un benchmark ingenuo con `System.nanoTime()` produce resultados poco fiables porque la JVM aplica optimizaciones que no se activarian en un escenario real de ejecucion. JMH se encarga de gestionar el warm-up, las iteraciones, la compilacion y la medicion para que los resultados reflejen el rendimiento real del codigo compilado y optimizado.

## Dependencias

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.37</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.37</version>
    <scope>test</scope>
</dependency>
```

En Gradle:

```groovy
testImplementation 'org.openjdk.jmh:jmh-core:1.37'
annotationProcessor 'org.openjdk.jmh:jmh-generator-annprocess:1.37'
```

## @Benchmark

La anotacion `@Benchmark` marca un metodo como punto de medicion. JMH ejecuta el metodo repetidamente durante las fases de warm-up y medicion.

```java
import org.openjdk.jmh.annotations.*;

public class MiBenchmark {

    @Benchmark
    public void medirStringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100; i++) {
            sb.append(i);
        }
        sb.toString();
    }
}
```

## @BenchmarkMode

Define las unidades de medida del benchmark. Los modos disponibles son:

- `Throughput`: operaciones por unidad de tiempo (ops/ms, ops/s).
- `AverageTime`: tiempo medio por operacion.
- `SampleTime`: distribucion estadistica de los tiempos.
- `SingleShotTime`: una sola ejecucion, util para medir cold start.
- `All`: todos los modos anteriores.

```java
@BenchmarkMode(Mode.Throughput)
@Benchmark
public void medirThroughput() { ... }
```

## @OutputTimeUnit

Especifica la unidad de tiempo para los resultados.

```java
@OutputTimeUnit(TimeUnit.MILLISECONDS)
```

## @State

Define el ambito de las variables de estado en el benchmark. Los tipos son:

- `Scope.Thread`: cada hilo tiene su propia instancia (la mas segura).
- `Scope.Benchmark`: compartida entre todos los hilos (simula concurrencia real).
- `Scope.Group`: compartida dentro de un grupo de hilos.

```java
@State(Scope.Thread)
public class EstadoLocal {
    String entrada = "Hola, mundo!";

    @Benchmark
    public int medirLongitud() {
        return entrada.length();
    }
}
```

## @Warmup

Configura la fase de calentamiento, donde la JVM compila y optimiza el codigo pero no se registran mediciones.

```java
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
```

## @Measurement

Configura la fase de medicion, donde los resultados si se registran.

```java
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
```

## @Fork

Especifica cuantos forks ejecutar. Cada fork arranca una nueva JVM, lo que evita la contaminacion entre ejecuciones.

```java
@Fork(value = 3, warmups = 1)
```

Un valor de 3 forks con 1 warmup significa: arrancar 4 JVMs, usar la primera solo para warm-up global, y las 3 restantes para medicion.

## Blackhole

JMH utiliza `Blackhole` para consumir valores sin que el JIT los elimine por considerar que no se usan. Si un metodo devuelve un valor y el benchmark no lo utiliza, el JIT puede eliminar la computacion porque detecta que el resultado es irrelevante.

```java
@Benchmark
public void medirCalculo(Blackhole agujero) {
    int resultado = calcular();
    agujero.consume(resultado);
}
```

Blackhole tambien puede consumir multiples valores y simular branches para evitar que el JIT elimine codigo muerto.

JMH es la herramienta de referencia para microbenchmarks en Java. Su uso requiere cuidado: es facil escribir benchmarks que midan algo distinto de lo que se pretende. La regla de oro es medir solo operaciones pequenas y aisladas, y siempre validar que los resultados sean coherentes con la logica del codigo.
