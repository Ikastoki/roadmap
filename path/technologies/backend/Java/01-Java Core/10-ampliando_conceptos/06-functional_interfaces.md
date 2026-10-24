# Functional Interfaces

Una interfaz funcional es una interfaz que contiene exactamente un metodo abstracto. Son la base de las expresiones lambda en Java: cualquier lambda puede asignarse a una interfaz funcional. El paquete `java.util.function` proporciona las interfaces funcionales mas comunes para la programacion funcional en Java.

## @FunctionalInterface

La anotacion `@FunctionalInterface` marca una interfaz como funcional. El compilador verifica que la interfaz tenga exactamente un metodo abstracto.

```java
@FunctionalInterface
public interface Comparador<T> {
    int comparar(T a, T b);
}
```

Si se anade un segundo metodo abstracto, el compilador lanza un error. Los metodos `default`, `static` y los heredados de `Object` no cuentan.

```java
@FunctionalInterface
public interface Validador<T> {
    boolean validar(T valor);
    // boolean otroMetodo(); // ERROR
    default void log() {}         // OK: default method
    static void info() {}         // OK: static method
}
```

## SAM (Single Abstract Method)

SAM es el acronimo de Single Abstract Method. Toda interfaz funcional es una SAM. Las expresiones lambda y las referencias a metodos son implementaciones concisas de SAM.

```java
// Expresion lambda que implementa la SAM
Comparador<Integer> c = (a, b) -> Integer.compare(a, b);
// Referencia a metodo
Comparador<Integer> c2 = Integer::compare;
```

## Predicate\<T\>

Representa una funcion booleana de un argumento. Su metodo abstracto es `test(T t)`.

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

```java
Predicate<String> noVacio = s -> s != null && !s.isEmpty();
Predicate<String> longitudMayor3 = s -> s.length() > 3;

System.out.println(noVacio.test("Hola")); // true

// Metodos por defecto: and, or, negate
Predicate<String> compuesto = noVacio.and(longitudMayor3);
Predicate<String> negado = noVacio.negate();
```

```java
// Uso tipico con streams
List<String> nombres = Arrays.asList("Ana", "", "Juan", null, "Maria");
List<String> filtrados = nombres.stream()
    .filter(noVacio)
    .collect(Collectors.toList());
```

## Consumer\<T\>

Representa una operacion que acepta un argumento y no devuelve resultado. Su metodo abstracto es `accept(T t)`.

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

```java
Consumer<String> imprimir = s -> System.out.println(s);
Consumer<String> mayusculas = s -> System.out.println(s.toUpperCase());

imprimir.accept("Hola"); // Hola

// andThen: encadenar consumidores
Consumer<String> ambos = imprimir.andThen(mayusculas);
ambos.accept("Hola"); // Hola \n HOLA (aunque mayusculas sale mal, mira la ortografia... ej: HOLA)
```

```java
// Uso con forEach
lista.forEach(imprimir);
```

## Supplier\<T\>

Representa un proveedor de resultados. No acepta argumentos y devuelve un valor. Su metodo abstracto es `get()`.

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

```java
Supplier<Double> aleatorio = () -> Math.random();
Supplier<String> saludo = () -> "Hola Mundo";

System.out.println(aleatorio.get());
System.out.println(saludo.get());
```

```java
// Supplier para creacion perezosa (lazy initialization)
Supplier<Conexion> conexionSupplier = () -> crearConexionCostosa();

// La conexion solo se crea cuando se invoca get()
Conexion cx = conexionSupplier.get();
```

```java
// Uso con Optional
String valor = Optional.ofNullable(getValor())
    .orElseGet(() -> calcularValorPorDefecto());
```

## Function\<T, R\>

Representa una funcion que acepta un argumento de tipo `T` y devuelve un resultado de tipo `R`. Su metodo abstracto es `apply(T t)`.

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

```java
Function<String, Integer> longitud = s -> s.length();
Function<Integer, String> aString = Object::toString;

int len = longitud.apply("Hola"); // 4

// compose y andThen
Function<String, String> mayusPrimero = aString.compose(longitud);
String resultado = mayusPrimero.apply("Hola"); // "4" (aplica longitud, luego toString)

Function<String, String> primeroLong = longitud.andThen(aString);
String resultado2 = primeroLong.apply("Hola"); // "4"
```

```java
// Uso con streams
List<String> palabras = Arrays.asList("Java", "Python", "Rust");
List<Integer> longitudes = palabras.stream()
    .map(longitud)
    .collect(Collectors.toList());
```

## UnaryOperator\<T\>

Es un caso especial de `Function<T, T>` donde el tipo de entrada y salida es el mismo.

```java
UnaryOperator<String> mayusculas = String::toUpperCase;
UnaryOperator<Integer> cuadrado = x -> x * x;

System.out.println(mayusculas.apply("hola")); // "HOLA"
```

## BinaryOperator\<T\>

Es un caso especial de `BiFunction<T, T, T>` donde los dos argumentos y el resultado son del mismo tipo.

```java
BinaryOperator<Integer> suma = (a, b) -> a + b;
BinaryOperator<String> concatenar = (a, b) -> a + b;

System.out.println(suma.apply(3, 5)); // 8
```

```java
// Uso con reduce
List<Integer> numeros = Arrays.asList(1, 2, 3, 4, 5);
int total = numeros.stream().reduce(0, suma);
```

## BiFunction\<T, U, R\>

Acepta dos argumentos de tipos `T` y `U` y devuelve un resultado de tipo `R`.

```java
@FunctionalInterface
public interface BiFunction<T, U, R> {
    R apply(T t, U u);
}
```

```java
BiFunction<String, String, String> unir = (a, b) -> a + " " + b;
BiFunction<Integer, Integer, Double> dividir = (a, b) -> a / (double) b;

System.out.println(unir.apply("Hola", "Mundo")); // "Hola Mundo"
```

## BiConsumer\<T, U\>

Acepta dos argumentos y no devuelve resultado.

```java
BiConsumer<String, Integer> imprimirPar = (nombre, edad) ->
    System.out.println(nombre + ": " + edad);

imprimirPar.accept("Ana", 30);
```

```java
// Uso con Map.forEach
mapa.forEach(imprimirPar);
```

## BiPredicate\<T, U\>

Acepta dos argumentos y devuelve un boolean.

```java
BiPredicate<String, String> mismaLongitud = (a, b) -> a.length() == b.length();
System.out.println(mismaLongitud.test("Hola", "Moto")); // true
```

## Interfaces funcionales especializadas

Para tipos primitivos, evitando autoboxing:

```java
IntPredicate par = n -> n % 2 == 0;
IntFunction<String> aString = String::valueOf;
DoubleSupplier aleatorio = Math::random;
LongConsumer imprimirLong = System.out::println;
```

## Tabla resumen

| Interfaz | Metodo | Entrada | Salida |
|----------|--------|---------|--------|
| `Predicate<T>` | `test(T)` | T | boolean |
| `Consumer<T>` | `accept(T)` | T | void |
| `Supplier<T>` | `get()` | - | T |
| `Function<T,R>` | `apply(T)` | T | R |
| `UnaryOperator<T>` | `apply(T)` | T | T |
| `BinaryOperator<T>` | `apply(T,T)` | T, T | T |
| `BiFunction<T,U,R>` | `apply(T,U)` | T, U | R |
| `BiConsumer<T,U>` | `accept(T,U)` | T, U | void |
| `BiPredicate<T,U>` | `test(T,U)` | T, U | boolean |
| `IntPredicate` | `test(int)` | int | boolean |
| `IntFunction<R>` | `apply(int)` | int | R |
| `DoubleSupplier` | `getAsDouble()` | - | double |

## Creacion de interfaces funcionales propias

```java
@FunctionalInterface
public interface Transformador<T, R> {
    R transformar(T entrada);

    default Transformador<T, R> andThen(Transformador<R, R> despues) {
        return t -> despues.transformar(this.transformar(t));
    }
}
```

```java
Transformador<String, Integer> aLongitud = String::length;
Transformador<Integer, String> aTexto = Object::toString;
Transformador<String, String> composicion = aLongitud.andThen(aTexto);
```
