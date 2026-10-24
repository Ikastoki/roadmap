# Streams

Un stream es una secuencia de elementos que soporta operaciones agregadas sobre los datos. No es una estructura de datos: no almacena elementos. Los streams toman datos de una fuente (coleccion, array, generador), aplican una pipeline de operaciones y producen un resultado sin modificar la fuente original. Este modelo funcional permite escribir procesamiento de datos complejo de forma declarativa, paralelizable y libre de efectos secundarios.

## Creacion de streams

Existen multiples formas de obtener un stream segun la naturaleza de la fuente de datos:

```java
// Desde una coleccion
List<String> lista = Arrays.asList("a", "b", "c");
Stream<String> stream1 = lista.stream();
Stream<String> stream2 = lista.parallelStream(); // version paralela

// Desde un array
String[] array = {"a", "b", "c"};
Stream<String> stream3 = Arrays.stream(array);
Stream<String> stream4 = Arrays.stream(array, 0, 2); // con rango

// Metodo factory Stream.of()
Stream<String> stream5 = Stream.of("a", "b", "c");
Stream<Integer> stream6 = Stream.of(1, 2, 3, 4, 5);

// Stream vacio
Stream<String> streamVacio = Stream.empty();

// Stream infinito con iterate()
Stream<Integer> pares = Stream.iterate(0, n -> n + 2);
Stream<Integer> limitado = Stream.iterate(0, n -> n < 100, n -> n + 2);

// Stream infinito con generate()
Stream<Double> aleatorios = Stream.generate(Math::random);
Stream<String> constante = Stream.generate(() -> "valor");

// Stream desde Builder
Stream<String> construido = Stream.<String>builder()
    .add("uno")
    .add("dos")
    .add("tres")
    .build();
```

Los metodos `iterate` y `generate` producen streams infinitos. Siempre deben combinarse con una operacion de cortocircuito como `limit()` para evitar bucles infinitos.

```java
Stream.iterate(0, n -> n + 1)
    .filter(n -> n % 2 == 0)
    .limit(10)
    .forEach(System.out::println);
```

## Operaciones intermedias

Las operaciones intermedias transforman el stream y devuelven otro stream, permitiendo encadenamiento. Son _lazy_: no se ejecutan hasta que una operacion terminal las dispara.

### filter

Selecciona los elementos que cumplen un predicado:

```java
Stream.of(1, 2, 3, 4, 5, 6)
    .filter(n -> n % 2 == 0)  // 2, 4, 6
```

### map

Transforma cada elemento aplicando una funcion:

```java
Stream.of("hola", "mundo")
    .map(String::toUpperCase) // "HOLA", "MUNDO"
    .map(s -> s.length())     // 4, 5
```

### flatMap

Aplana estructuras anidadas. Cada elemento produce un stream y todos se concatenan en uno solo:

```java
List<List<Integer>> anidada = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4)
);
List<Integer> plana = anidada.stream()
    .flatMap(Collection::stream) // 1, 2, 3, 4
    .collect(Collectors.toList());
```

Es especialmente util para dividir y procesar cadenas:

```java
Stream.of("Hola mundo", "Java stream")
    .flatMap(s -> Arrays.stream(s.split(" ")))
    // "Hola", "mundo", "Java", "stream"
```

### sorted

Ordena los elementos segun un orden natural o un comparador:

```java
Stream.of(3, 1, 4, 1, 5)
    .sorted()                           // 1, 1, 3, 4, 5
    .sorted(Comparator.reverseOrder())  // 5, 4, 3, 1, 1
```

### distinct

Elimina elementos duplicados usando `equals()`:

```java
Stream.of(1, 2, 2, 3, 3, 3)
    .distinct() // 1, 2, 3
```

### peek

Permite inspeccionar elementos en medio de la pipeline sin modificar el stream. Util principalmente para depuracion:

```java
Stream.of("uno", "dos", "tres")
    .peek(s -> System.out.println("Entrada: " + s))
    .map(String::toUpperCase)
    .peek(s -> System.out.println("Salida: " + s))
    .collect(Collectors.toList());
```

> Aunque `peek` es util para depurar, no debe usarse para efectos secundarios en produccion. La API documenta que las operaciones intermedias son _non-interfering_ y _stateless_; `peek` con efectos secundarios viola esa expectativa.

## Operaciones terminales

Las operaciones terminales producen un resultado o efecto secundario y cierran el stream. Una vez ejecutadas, el stream no puede reutilizarse.

### collect

Es la operacion terminal mas potente. Acumula los elementos en una estructura de datos usando un `Collector`:

```java
List<String> lista = stream.collect(Collectors.toList());
Set<String> conjunto = stream.collect(Collectors.toSet());
Collection<String> coleccion = stream.collect(Collectors.toCollection(LinkedList::new));
```

### reduce

Combina los elementos en un unico valor mediante una funcion de acumulacion binaria:

```java
Optional<Integer> suma = numeros.stream().reduce(Integer::sum);
int sumaConIdentidad = numeros.stream().reduce(0, Integer::sum);

// reduce con tres parametros (para paralelismo)
int resultado = numeros.stream()
    .reduce(0, Integer::sum, Integer::sum);
```

### forEach

Ejecuta una accion para cada elemento. Es una operacion terminal con efectos secundarios:

```java
stream.forEach(System.out::println);
```

### count

Devuelve el numero de elementos como `long`:

```java
long total = stream.filter(s -> s.length() > 3).count();
```

### min / max

Devuelven el elemento minimo o maximo segun un comparador, envuelto en `Optional`:

```java
Optional<String> menor = stream.min(Comparator.naturalOrder());
Optional<String> mayor = stream.max(String::compareToIgnoreCase);
```

### anyMatch / allMatch / noneMatch

Operaciones de cortocircuito que evaluan un predicado sobre los elementos:

```java
boolean algunPar = numeros.stream().anyMatch(n -> n % 2 == 0);
boolean todosPares = numeros.stream().allMatch(n -> n % 2 == 0);
boolean ningunPar = numeros.stream().noneMatch(n -> n % 2 == 0);
```

Son _short-circuiting_: pueden devolver resultado sin recorrer todo el stream.

## Collectors

La clase `Collectors` proporciona implementaciones listas para las operaciones de recoleccion mas habituales.

### toList / toSet / toCollection

```java
List<String> lista = stream.collect(Collectors.toList());
Set<String> conjunto = stream.collect(Collectors.toSet());
TreeSet<String> arbol = stream.collect(Collectors.toCollection(TreeSet::new));
```

### joining

Concatena cadenas con un delimitador opcional:

```java
String unido = stream.collect(Collectors.joining());
String conComas = stream.collect(Collectors.joining(", "));
String conFormato = stream.collect(Collectors.joining(", ", "[", "]"));
```

### groupingBy

Agrupa elementos segun una funcion clasificadora, produciendo un `Map<K, List<V>>`:

```java
Map<Integer, List<String>> porLongitud = palabras.stream()
    .collect(Collectors.groupingBy(String::length));

Map<Integer, Set<String>> porLongitudSet = palabras.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.toSet()));

Map<Integer, Long> countPorLongitud = palabras.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));
```

### partitioningBy

Particiona en dos grupos segun un predicado (true/false):

```java
Map<Boolean, List<Integer>> particion = numeros.stream()
    .collect(Collectors.partitioningBy(n -> n > 10));
```

### mapping y reducing

```java
// mapping: transforma antes de recolectar
Map<Integer, List<String>> resultado = palabras.stream()
    .collect(Collectors.groupingBy(String::length,
        Collectors.mapping(String::toUpperCase, Collectors.toList())));

// reducing: reduce dentro de un colector
Map<String, Optional<Integer>> maxPorGrupo = datos.stream()
    .collect(Collectors.groupingBy(
        Dato::categoria,
        Collectors.reducing(0, Dato::valor, Integer::max)));
```

## Parallel streams

Los streams pueden ejecutarse en paralelo sin modificar la logica de la pipeline, simplemente usando `parallelStream()` en lugar de `stream()` o llamando a `.parallel()` sobre un stream existente:

```java
List<Integer> numeros = IntStream.rangeClosed(1, 10_000_000)
    .boxed()
    .collect(Collectors.toList());

int sumaParalela = numeros.parallelStream()
    .reduce(0, Integer::sum);
```

El framework Fork/Join de Java divide automaticamente la fuente de datos en segmentos, los procesa en multiples hilos, y combina los resultados parciales. Sin embargo, el paralelismo no es gratuito. Para fuentes pequenas, el overhead de particion y sincronizacion puede superar la ganancia.

> Los parallel streams son efectivos cuando: la fuente de datos es grande, las operaciones son independientes entre elementos (stateless), la funcion de combinacion es asociativa, y no hay bloqueos o contención de E/S. Usar parallel streams con operaciones que comparten estado mutable produce resultados incorrectos.

Para garantizar correccion en paralelo, las operaciones deben ser _stateless_ y _non-interfering_. El siguiente codigo es incorrecto:

```java
List<Integer> resultado = new ArrayList<>(); // estado mutable compartido
numeros.parallelStream()
    .filter(n -> n % 2 == 0)
    .forEach(n -> resultado.add(n)); // race condition!
```

La version correcta delega la recoleccion al framework:

```java
List<Integer> resultado = numeros.parallelStream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
```
