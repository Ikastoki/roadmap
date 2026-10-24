# Benchmarks en testing

Los benchmarks y los tests son herramientas diferentes con objetivos distintos. Mezclarlos o confundirlos conduce a malas practicas y a interpretaciones erroneas del rendimiento.

## Tests vs Benchmarks

Un test verifica que el codigo se comporta correctamente: dado un input, produce el output esperado. Un benchmark mide cuanto tarda en producirse ese output. Son preguntas ortogonales. Un metodo puede ser correcto y lento, o rapido e incorrecto.

```java
// Test
@Test
void ordenarFunciona() {
    List<Integer> resultado = ordenar(List.of(3, 1, 2));
    assertThat(resultado).containsExactly(1, 2, 3);
}

// Benchmark
@Benchmark
public void ordenarRendimiento() {
    ordenar(List.of(3, 1, 2));
}
```

Un test debe ser rapido para fomentar su ejecucion frecuente. Un benchmark, por su naturaleza, es lento y costoso. Ejecutar benchmarks en cada commit no es practico.

## Cuando usar JMH vs JUnit

Usar JUnit para medir rendimiento es un error frecuente. Sin warm-up, sin control del JIT y sin gestion de forks, los resultados son ruido. JUnit mide correccion, no velocidad.

JMH debe usarse cuando:

- Se compara el rendimiento de dos implementaciones del mismo algoritmo.
- Se evalua el impacto de una optimizacion.
- Se caracteriza el comportamiento de una operacion bajo diferentes tamanos de entrada.
- Se detectan regresiones de rendimiento en componentes criticos.

JUnit (o cualquier framework de testing) se usa para la verificacion funcional. Las pruebas de rendimiento no deben interferir con el ciclo de test rapido.

## Regresiones de rendimiento

Una regresion de rendimiento ocurre cuando un cambio en el codigo degrada la velocidad de ejecucion sin alterar la correccion. Detectarlas requiere un sistema de benchmark automatizado que compare resultados contra una linea base.

El flujo tipico es:

1. Establecer una linea base ejecutando los benchmarks en la rama principal.
2. Al introducir un cambio, ejecutar los mismos benchmarks en la rama de feature.
3. Comparar resultados y decidir si la diferencia es estadisticamente significativa.

```bash
# Rama principal
java -jar benchmarks.jar -rf json -rff baseline.json

# Rama feature
java -jar benchmarks.jar -rf json -rff feature.json
```

Herramientas como `jmh-report` o scripts de analisis permiten comparar automaticamente los resultados y fallar el build si la regresion supera un umbral (por ejemplo, 10 % mas lento).

> Comparar benchmarks entre entornos distintos (local vs CI, portatil vs servidor) no es fiable. Los resultados solo son comparables dentro del mismo entorno hardware y software.

## Cuando no usar benchmarks

No todo necesita un benchmark. La mayoria del codigo de aplicacion no tiene requisitos de rendimiento tan estrictos que justifiquen el costo de mantenerlos. Anadir benchmarks sin criterio incrementa el tiempo de build, el ruido en las metricas y la complejidad del proyecto.

Los benchmarks se reservan para:

- Colecciones y algoritmos de uso intensivo.
- Operaciones de I/O (serializacion, compresion, cifrado).
- Componentes del core del framework o libreria.
- Codigo donde cada microsegundo cuenta (trading, gaming, streaming).

Para el resto, el rendimiento se evalua con pruebas de integracion y carga, no con microbenchmarks. La regla es: si no puedes justificar el benchmark con una metrica de negocio, probablemente no lo necesitas.
