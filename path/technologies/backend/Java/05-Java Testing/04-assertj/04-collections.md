# Collections

Las aserciones para colecciones en AssertJ constituyen una de las areas mas potentes de la biblioteca. A traves de `AbstractIterableAssert` y sus subtipos (`ListAssert`, `SetAssert`, etc.), es posible verificar tamaño, contenido, orden, propiedades de los elementos y realizar proyecciones funcionales sin necesidad de bucles manuales ni estructuras auxiliares.

## isEmpty / isNotEmpty

Verifican si la coleccion esta vacia consultando `isEmpty()` del propio objeto. Son semanticamente equivalentes a `assertThat(collection.size()).isEqualTo(0)` pero con mensajes de error mas claros.

```java
List<String> listaVacia = new ArrayList<>();
assertThat(listaVacia).isEmpty();

List<String> listaConElementos = Arrays.asList("a", "b");
assertThat(listaConElementos).isNotEmpty();
```

## contains / doesNotContain / containsOnlyOnce

`contains` verifica que todos los elementos especificados esten presentes en la coleccion, sin importar el orden ni la presencia de elementos adicionales. `doesNotContain` asegura la ausencia total de ciertos valores. `containsOnlyOnce` verifica que cada elemento aparezca exactamente una vez.

```java
List<String> colores = Arrays.asList("rojo", "verde", "azul", "rojo");

assertThat(colores).contains("rojo", "verde");
assertThat(colores).doesNotContain("negro");
assertThat(colores).containsOnlyOnce("verde");
assertThat(colores).containsOnlyOnce("rojo"); // falla: aparece dos veces
```

## containsExactly / containsExactlyInAnyOrder

`containsExactly` verifica que la coleccion contenga exactamente los elementos especificados, en el mismo orden y sin elementos adicionales ni faltantes. Es la asercion mas estricta para listas ordenadas.

`containsExactlyInAnyOrder` relaja la restriccion de orden: los elementos deben ser exactamente los mismos, pero pueden aparecer en cualquier secuencia.

```java
List<String> numeros = Arrays.asList("uno", "dos", "tres");

assertThat(numeros).containsExactly("uno", "dos", "tres");
assertThat(numeros).containsExactlyInAnyOrder("tres", "uno", "dos");
assertThat(numeros).containsExactly("uno", "tres"); // falla: falta "dos"
```

> La diferencia entre `contains` y `containsExactly` es fundamental. `contains` admite elementos adicionales en la coleccion; `containsExactly` exige una coincidencia perfecta tanto en contenido como en cantidad y (en su variante sin `InAnyOrder`) en orden.

## hasSize / hasSizeLessThan / hasSizeGreaterThan

`hasSize` compara el tamaño de la coleccion con un entero. AssertJ ofrece ademas comparadores relacionales para validar rangos de tamaño.

```java
List<String> items = Arrays.asList("a", "b", "c", "d");

assertThat(items).hasSize(4);
assertThat(items).hasSizeGreaterThan(2);
assertThat(items).hasSizeLessThan(10);
assertThat(items).hasSizeBetween(1, 5);
```

## filteredOn

`filteredOn` aplica un predicado o una condicion de campo sobre la coleccion antes de ejecutar la asercion. El resultado es una nueva proyeccion filtrada sobre la que se encadenan mas verificaciones. Acepta tanto lambdas como expresiones de tipo `"campo == valor"`.

```java
List<Persona> personas = Arrays.asList(
    new Persona("Ana", 30),
    new Persona("Luis", 17),
    new Persona("Eva", 25)
);

assertThat(personas)
    .filteredOn(p -> p.getEdad() >= 18)
    .hasSize(2);

assertThat(personas)
    .filteredOn("nombre", "Ana")
    .hasSize(1);
```

> La sintaxis `filteredOn("campo", "valor")` utiliza reflexion para acceder a la propiedad, por lo que es menos segura en tiempo de compilacion que la version con lambda. Sin embargo, resulta mas legible para filtros simples.

## extracting

`extracting` proyecta la coleccion original extrayendo una propiedad de cada elemento, produciendo una nueva lista con los valores extraidos sobre la que se continua encadenando aserciones. Es especialmente util cuando solo interesa verificar ciertos atributos de los objetos.

```java
List<Persona> personas = Arrays.asList(
    new Persona("Ana", 30),
    new Persona("Luis", 17)
);

assertThat(personas)
    .extracting(Persona::getNombre)
    .containsExactly("Ana", "Luis");

// Extraccion multiple (tupla)
assertThat(personas)
    .extracting(Persona::getNombre, Persona::getEdad)
    .containsExactly(
        tuple("Ana", 30),
        tuple("Luis", 17)
    );
```

## containsKey / containsEntry (Mapas)

Para aserciones sobre `Map`, AssertJ proporciona `MapAssert` con metodos especificos para claves, valores y entradas.

```java
Map<String, Integer> edades = new HashMap<>();
edades.put("Ana", 30);
edades.put("Luis", 17);

assertThat(edades).containsKey("Ana");
assertThat(edades).doesNotContainKey("Pedro");
assertThat(edades).containsEntry("Ana", 30);
assertThat(edades).doesNotContainEntry("Luis", 25);
```

Otras aserciones utiles para mapas incluyen `containsKeys`, `containsValues`, `containsOnlyKeys`, y `hasSize`, todas ellas encadenables y con mensajes de error que muestran el contenido completo del mapa cuando fallan.

```java
assertThat(edades)
    .hasSize(2)
    .containsKeys("Ana", "Luis")
    .containsValues(30, 17);
```
