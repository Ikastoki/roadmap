# jqwik

jqwik es una libreria de property-based testing para Java 17+ que se integra directamente con el motor de JUnit 5. A diferencia de otras alternativas, jqwik no requiere un runner separado ni una configuracion compleja: funciona como una extension mas de JUnit 5, usando el mecanismo de `TestEngine` para ejecutar metodos anotados con `@Property`.

## Dependencias

Para incorporar jqwik en un proyecto Maven se anade la dependencia del motor de test:

```xml
<dependency>
    <groupId>net.jqwik</groupId>
    <artifactId>jqwik</artifactId>
    <version>1.8.4</version>
    <scope>test</scope>
</dependency>
```

En Gradle:

```groovy
testImplementation 'net.jqwik:jqwik:1.8.4'
```

La dependencia incluye el motor de jqwik, que se registra automaticamente mediante el servicio `TestEngine` de JUnit 5. No se necesita ninguna configuracion adicional.

## @Property

La anotacion `@Property` marca un metodo como test basado en propiedades. Acepta parametros de configuracion como el numero de intentos, el criterio de filtrado y el modo de shrinking.

```java
@Property(tries = 1000, shrinking = ShrinkingMode.FULL)
boolean cualquierNombre(@ForAll int valor) {
    return Math.abs(valor) >= 0;
}
```

El parametro `tries` indica cuantos valores generados se probaran antes de dar la propiedad por valida. Por defecto son 1000.

## @ForAll y generadores

Los parametros anotados con `@ForAll` reciben valores generados automaticamente. jqwik incluye generadores para todos los tipos primitivos, String, List, Map, Set, Optional, Stream, arrays y combinaciones de estos.

Se puede restringir el dominio mediante anotaciones de configuracion:

```java
@Property
boolean numerosPositivos(@ForAll @IntRange(min = 1, max = 10000) int n) {
    return procesar(n) > 0;
}
```

## @StringRange y @StringLength

Para cadenas, jqwik ofrece anotaciones especificas que controlan la longitud y los caracteres permitidos.

```java
@Property
boolean cadenasConPrefijo(@ForAll @StringLength(min = 1, max = 50) @AlphaChars String texto) {
    String resultado = procesarTexto(texto);
    return resultado.startsWith("PREF_");
}
```

`@AlphaChars` limita los caracteres a letras. Existen tambien `@NumericChars`, `@LowerCaseChars`, `@UpperCaseChars` y `@Whitespace`.

## Shrinking

Cuando una propiedad falla, jqwik aplica un algoritmo de shrinking para encontrar el contraejemplo mas simple. El modo de shrinking se configura en `@Property`:

- `FULL`: shrinking exhaustivo, mas lento pero encuentra el minimo absoluto.
- `BOUNDED`: compromiso entre velocidad y minimalidad.
- `OFF`: sin shrinking, util para depurar el generador.

```java
@Property(shrinking = ShrinkingMode.BOUNDED)
boolean noNulos(@ForAll String s) {
    return s != null;
}
```

## @EdgeCases

Los casos extremos son valores que tienen alta probabilidad de causar errores: valores nulos, cadenas vacias, cero, valores maximos y minimos. jqwik los anade automaticamente a la generacion aleatoria.

Se pueden configurar o deshabilitar mediante `@EdgeCases`:

```java
@Property
boolean conCasosExtremos(@ForAll @IntRange(min = -100, max = 100) int n,
                          @ForAll @EdgeCases(edgeCases = EdgeCasesMode.MIXIN) String s) {
    return metodo(n, s) != null;
}
```

Los modos disponibles son `MIXIN` (por defecto), `FIRST` (ejecutar antes de los aleatorios) y `NONE` (desactivar).

## Propiedades con multiples parametros

Se pueden combinar varios generadores en una misma propiedad. jqwik genera todas las combinaciones de forma independiente.

```java
@Property
boolean ordenamiento(@ForAll List<@IntRange(min = -1000, max = 1000) Integer> lista) {
    List<Integer> ordenada = ordenar(lista);
    return ordenada.stream().sorted().toList().equals(ordenada);
}
```

jqwik es una herramienta madura, bien documentada y con un modelo mental cercano al de JUnit 5. Para equipos que ya usan JUnit 5, la curva de aprendizaje es minima y la integracion inmediata.
