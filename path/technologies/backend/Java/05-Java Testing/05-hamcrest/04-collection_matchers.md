# Collection matchers

Los matchers para colecciones en Hamcrest permiten verificar contenido, tamaño, ausencia y propiedades de los elementos contenidos en `Collection`, `List`, `Set`, `Map` y arrays. Se definen principalmente en `org.hamcrest.Matchers` y en `org.hamcrest.collection`.

## hasItem / hasItems

`hasItem` verifica que la coleccion contenga al menos una ocurrencia del elemento especificado. Acepta tanto un valor literal como un matcher para busqueda por condicion.

`hasItems` es la version varargs: verifica que la coleccion contenga todos los elementos especificados, pero no exige que sean los unicos ni que respeten un orden.

```java
List<String> nombres = Arrays.asList("Ana", "Luis", "Eva");

assertThat(nombres, hasItem("Ana"));
assertThat(nombres, hasItems("Ana", "Eva"));
assertThat(nombres, hasItem(startsWith("L")));
assertThat(nombres, not(hasItem("Pedro")));
```

> `hasItem` y `hasItems` no verifican orden ni cardinalidad exacta. Si se necesita comprobar que un elemento aparece exactamente una vez o que no hay elementos adicionales, deben usarse matchers mas estrictos como `contains`.

## hasSize

`hasSize` verifica el tamaño de la coleccion. El matcher acepta un entero o un `Matcher<? super Integer>` para validaciones relacionales.

```java
assertThat(nombres, hasSize(3));
assertThat(nombres, hasSize(greaterThan(1)));
assertThat(nombres, hasSize(lessThan(10)));
```

Cuando falla, el mensaje de error muestra el tamaño real y el esperado, incluyendo el contenido completo de la coleccion si es relevante.

## empty / emptyCollection / emptyIterable

`empty` y sus alias (`emptyCollection`, `emptyIterable`) verifican que la coleccion no contenga elementos.

```java
List<String> vacia = new ArrayList<>();
assertThat(vacia, empty());
assertThat(vacia, emptyCollection());
assertThat(Arrays.asList("a"), not(empty()));
```

Para arrays, existe `emptyArray()`:

```java
String[] arrayVacio = new String[0];
assertThat(arrayVacio, emptyArray());
```

## contains / containsInAnyOrder

`contains` con argumentos variables verifica que la coleccion contenga exactamente los elementos especificados en el mismo orden, sin elementos adicionales ni faltantes. Es el matcher mas estricto.

`containsInAnyOrder` relaja la restriccion de orden: los elementos deben ser exactamente los mismos, pero el orden es irrelevante.

```java
List<String> numeros = Arrays.asList("uno", "dos", "tres");

assertThat(numeros, contains("uno", "dos", "tres"));
assertThat(numeros, containsInAnyOrder("tres", "uno", "dos"));
assertThat(numeros, not(contains("uno", "tres")));  // falta "dos"
```

Ambos metodos aceptan matchers como argumentos, permitiendo condiciones sobre cada elemento:

```java
assertThat(numeros, contains(
    startsWith("u"),
    startsWith("d"),
    startsWith("t")
));
```

## hasEntry / hasKey / hasValue (Mapas)

Para aserciones sobre `Map`, Hamcrest proporciona tres matchers especificos.

`hasEntry` verifica que el mapa contenga una entrada con una clave y un valor concretos. `hasKey` verifica solo la existencia de la clave. `hasValue` verifica solo la existencia del valor.

```java
Map<String, Integer> edades = new HashMap<>();
edades.put("Ana", 30);
edades.put("Luis", 17);

assertThat(edades, hasEntry("Ana", 30));
assertThat(edades, hasKey("Luis"));
assertThat(edades, hasValue(17));
assertThat(edades, not(hasEntry("Ana", 25)));
```

> Los matchers `hasEntry`, `hasKey` y `hasValue` tambien aceptan matchers como argumentos, permitiendo busquedas parciales. Por ejemplo, `hasEntry(startsWith("A"), greaterThan(20))` verifica que exista una entrada cuya clave empiece por "A" y cuyo valor sea mayor que 20.

```java
assertThat(edades, hasEntry(startsWith("A"), greaterThan(25)));
assertThat(edades, hasKey(containsString("An")));
```

Para colecciones de objetos, `everyItem` aplica un matcher a todos los elementos de la coleccion:

```java
List<Integer> numeros = Arrays.asList(1, 2, 3, 4, 5);
assertThat(numeros, everyItem(greaterThan(0)));
```

Y `hasItemInArray` para arrays tradicionales:

```java
String[] array = {"a", "b", "c"};
assertThat(array, hasItemInArray("b"));
```
