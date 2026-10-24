# Combinators

La capacidad de composicion logica es el rasgo distintivo de Hamcrest frente a otras bibliotecas de aserciones. Hamcrest permite combinar matchers mediante operadores logicos estandar (`and`, `or`, `not`) y a traves de metodos de azucar sintactico (`both`, `either`) que mejoran la legibilidad de las condiciones compuestas. Ademas, `everyItem` extiende la composicion a colecciones completas.

## allOf

`allOf` exige que todos los matchers proporcionados se cumplan simultaneamente. Es el equivalente logico del operador `&&` (AND booleano). Si alguno de los matchers falla, la asercion completa falla.

```java
String texto = "Hola Mundo desde Hamcrest";

assertThat(texto, allOf(
    containsString("Hola"),
    containsString("Mundo"),
    containsString("Hamcrest")
));

assertThat(42, allOf(
    greaterThan(10),
    lessThan(100),
    not(equalTo(0))
));
```

Cuando falla, el mensaje de error lista cada matcher y senala cual de ellos no se cumplio, facilitando la identificacion del problema.

## anyOf

`anyOf` exige que al menos uno de los matchers se cumpla. Es el equivalente logico del operador `||` (OR booleano). Si ningun matcher se cumple, la asercion falla mostrando todos los intentos fallidos.

```java
assertThat("ERROR", anyOf(
    containsString("ERROR"),
    containsString("WARN"),
    containsString("FATAL")
));

assertThat(100, anyOf(
    equalTo(100),
    equalTo(200),
    equalTo(300)
));
```

> Una diferencia practica entre `allOf` y `anyOf` es el comportamiento en los mensajes de error. `allOf` muestra solo el primer matcher que falla (short-circuit). `anyOf` muestra todos los matchers que fallaron porque necesita evaluarlos todos para determinar si alguno tuvo exito.

## both / and

`both` es azucar sintactico para `allOf` que permite una sintaxis mas cercana al lenguaje natural mediante encadenamiento de metodos. Devuelve un objeto intermedio sobre el que se invoca `and` para agregar condiciones.

```java
assertThat("Hola Mundo", both(containsString("Hola")).and(containsString("Mundo")));
assertThat(50, both(greaterThan(10)).and(lessThan(100)));
```

Es semanticamente identico a `allOf`, pero la sintaxis suele leerse mejor en aserciones con solo dos condiciones.

## either / or

`either` es el equivalente de `both` pero para `anyOf`. Permite expresar condiciones alternativas de forma mas legible.

```java
assertThat("ERROR", either(containsString("ERROR")).or(containsString("FATAL")));
assertThat(200, either(equalTo(200)).or(equalTo(404)).or(equalTo(500)));
```

Al igual que `both`, `either` es semanticamente equivalente a `anyOf`, pero su sintaxis es preferible cuando se combinan solo dos o tres alternativas.

## not

`not` niega el matcher que recibe. Puede aplicarse sobre cualquier matcher, incluyendo matchers compuestos, para expresar la condicion contraria.

```java
assertThat("Hola", not("Adios"));
assertThat("Hola", not(containsString("x")));
assertThat(Arrays.asList(1, 2, 3), not(empty()));

// Negacion de composiciones
assertThat("Hola", not(both(containsString("H")).and(containsString("x"))));
assertThat(50, not(anyOf(greaterThan(100), lessThan(10))));
```

> Usar `not` con matchers compuestos requiere atencion a las leyes de De Morgan: `not(allOf(a, b))` equivale a `anyOf(not(a), not(b))`. Elegir una forma u otra depende de que expresion resulte mas legible en cada contexto.

## everyItem

`everyItem` aplica un matcher a cada elemento de una coleccion o iterable. La asercion pasa si y solo si todos los elementos cumplen el matcher. Es el equivalente funcional de `allMatch` de Java Streams.

```java
List<Integer> positivos = Arrays.asList(1, 2, 3, 4, 5);
assertThat(positivos, everyItem(greaterThan(0)));

List<String> noVacios = Arrays.asList("a", "b", "c");
assertThat(noVacios, everyItem(not(emptyString())));

// Composicion con matchers de objetos
assertThat(personas, everyItem(hasProperty("edad", greaterThan(18))));
```

Cuando la coleccion esta vacia, `everyItem` pasa por definicion (vacuidad). Si se necesita verificar que la coleccion no este vacia y que todos los elementos cumplan la condicion, deben combinarse ambos matchers:

```java
assertThat(lista, both(not(empty())).and(everyItem(greaterThan(0))));
```

> `everyItem` es especialmente util en tests de validacion de datos, donde se espera que todos los elementos de una coleccion cumplan una invariante. Combina bien con `hasProperty` para verificar atributos de objetos dentro de colecciones.
