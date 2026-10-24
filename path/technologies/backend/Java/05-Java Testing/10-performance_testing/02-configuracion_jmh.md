# Configuracion de JMH

La configuracion de un benchmark con JMH puede realizarse mediante anotaciones o de forma programatica. La combinacion de ambas permite ajustar el comportamiento para obtener mediciones precisas y repetibles.

## Iteraciones

Las iteraciones se dividen en dos fases: warm-up y medicion. Durante el warm-up la JVM compila y optimiza el codigo; el recuento de iteraciones debe ser suficiente para que la curva de rendimiento se estabilice antes de empezar a medir.

```java
@Warmup(iterations = 5, time = 2, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
```

En general, 3-5 iteraciones de warm-up y 5-10 de medicion son adecuadas para la mayoria de los benchmarks. Si el codigo tiene un perfil de ejecucion muy variable (por GC, class loading, etc.), pueden necesitarse mas iteraciones.

## Threads

Por defecto, JMH ejecuta un hilo por benchmark. Se puede configurar el numero de hilos para simular concurrencia real o para detectar problemas de contención.

```java
@Threads(4)
```

El uso de multiples hilos debe hacerse con cuidado: si el codigo bajo prueba no es thread-safe, los resultados seran invalidos.

## Forks

El fork controla cuantas JVMs independientes se arrancan para la medicion. Cada fork limpia el estado de la JVM, incluyendo el JIT, la memoria y los caches.

```java
@Fork(value = 3, warmups = 1)
```

Sin forks, los resultados de distintas ejecuciones pueden contaminarse entre si. Con forks, cada medicion parte de cero. El costo es el tiempo de arranque de cada JVM.

## Profilers

JMH incluye profilers integrados que recolectan informacion adicional durante la ejecucion. Se configuran mediante la anotacion `@Fork` con el parametro `jvmArgsAppend` o mediante la linea de comandos.

### GC Profiler

Mide la actividad del garbage collector durante el benchmark. Revela si el codigo bajo prueba genera residuos que afectan al rendimiento.

```bash
java -jar benchmarks.jar -prof gc
```

El profiler GC muestra el numero de colecciones, el tiempo invertido y la memoria asignada por operacion.

### Stack Profiler

Captura muestras del stack trace para identificar donde se invierte el tiempo de CPU.

```bash
java -jar benchmarks.jar -prof stack
```

### Async Profiler

En Linux, se puede integrar con async-profiler para obtener perfiles de CPU, allocacion y locks con minimo overhead.

## Output format

JMH soporta multiples formatos de salida para integrarse con herramientas de visualizacion y CI.

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
```

Los resultados se pueden exportar a JSON, CSV o texto plano.

```bash
java -jar benchmarks.jar -rf json -rff resultados.json
```

## Ejecutar benchmarks

Los benchmarks se empaquetan como un JAR ejecutable mediante el plugin de Maven:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>org.openjdk.jmh.Main</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
mvn clean package
java -jar target/benchmarks.jar
```

En Gradle existe el plugin `gradle-jmh-plugin`:

```groovy
plugins {
    id 'me.champeau.jmh' version '0.7.2'
}

jmh {
    warmupIterations = 5
    iterations = 10
    fork = 3
}
```

```bash
./gradlew jmh
```

La configuracion de JMH debe documentarse junto con los resultados. Sin el contexto de las iteraciones, forks y modos usados, los numeros carecen de significado. Un benchmark que no especifica su configuracion no es reproducible y, por tanto, no es fiable.
