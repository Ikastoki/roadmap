# Soft assertions

Las aserciones tradicionales en JUnit y AssertJ siguen el patron de _fail-fast_: en cuanto una asercion falla, se lanza una `AssertionError` y el test termina inmediatamente, impidiendo que se ejecuten las aserciones restantes. Esto obliga a corregir los fallos uno por uno en ciclos sucesivos de prueba-error. Las _soft assertions_ de AssertJ resuelven este problema acumulando todos los fallos y reportandolos conjuntamente al final del bloque.

## SoftAssertionsProvider

La interfaz `SoftAssertionsProvider` define el contrato para las aserciones suaves. Su implementacion principal es `SoftAssertions`, una clase que recolecta en una estructura interna todos los errores producidos durante la ejecucion de las aserciones y los reporta juntos cuando se invoca `assertAll()`.

```java
SoftAssertions softly = new SoftAssertions();
softly.assertThat("Hola").isEqualTo("Adios");
softly.assertThat(42).isEqualTo(100);
softly.assertAll(); // lanza un error con ambos fallos
```

Sin `softly.assertAll()`, el test pasaria incluso si hubo fallos, porque los errores quedan almacenados internamente sin ser lanzados. Es responsabilidad del desarrollador invocar `assertAll()` al final del bloque para materializar los fallos.

## assertThatSoftly

AssertJ proporciona `assertThatSoftly` como metodo estatico que automatiza la llamada a `assertAll()`. Recibe un bloque lambda con un parametro `SoftAssertions` y ejecuta automaticamente la verificacion al salir del bloque, eliminando el riesgo de olvidar `assertAll()`.

```java
assertThatSoftly(softly -> {
    softly.assertThat("Hola").isEqualTo("Adios");
    softly.assertThat(42).isEqualTo(100);
});
```

La ventaja es doble: no se necesita instanciar `SoftAssertions` manualmente ni recordar invocar `assertAll()`. El bloque lambda actua como ambito de recoleccion, y al terminar se lanza un `SoftAssertionError` que agrupa todas las violaciones.

## Reporta todos los failures, no falla en primera

La diferencia fundamental entre una asercion dura y una blanda es el momento de notificacion del fallo:

```java
// Asercion dura: falla en la primera linea
assertThat("Hola").isEqualTo("Adios");  // test termina aqui
assertThat(42).isEqualTo(100);           // nunca se ejecuta

// Asercion blanda: continua ejecutando
assertThatSoftly(softly -> {
    softly.assertThat("Hola").isEqualTo("Adios"); // registra fallo
    softly.assertThat(42).isEqualTo(100);          // registra fallo
    // al salir del lambda, lanza ambos fallos juntos
});
```

Cuando `assertAll()` encuentra uno o mas fallos, la excepcion `SoftAssertionError` incluye un mensaje que lista cada uno de los errores individuales con su descripcion completa, permitiendo al desarrollador conocer de un vistazo todos los problemas sin necesidad de ejecutar el test repetidamente.

```
Multiple Failures (2 failures)
-- failure 1 --
expected: "Adios" but was: "Hola"
-- failure 2 --
expected: 100 but was: 42
```

> Las soft assertions son especialmente utiles en tests de integracion o aceptacion donde un solo objeto debe cumplir multiples invariantes. En lugar de abortar en el primer incumplimiento, se recolectan todas las violaciones y se presentan juntas. Sin embargo, en tests unitarios puros donde cada asercion verifica una condicion independiente, las aserciones duras suelen ser preferibles porque son mas directas y no requieren envoltura adicional.

Tambien es posible mezclar aserciones duras y blandas dentro del mismo test. Las aserciones duras fuera del `assertThatSoftly` siguen el comportamiento fail-fast, mientras que las incluidas en el bloque se acumulan:

```java
assertThat(condicionPrevia).isTrue(); // fail-fast si falla
assertThatSoftly(softly -> {
    softly.assertThat(propiedad1).isEqualTo(esperado1);
    softly.assertThat(propiedad2).isEqualTo(esperado2);
});
assertThat(condicionPosterior).isFalse(); // fail-fast
```
