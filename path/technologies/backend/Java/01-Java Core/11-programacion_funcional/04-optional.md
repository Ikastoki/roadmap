# Optional

`Optional<T>` es un contenedor que puede contener un valor no nulo o ningun valor. Fue introducido en Java 8 para reducir la necesidad de comprobaciones explicitas de null y para expresar en el sistema de tipos la posibilidad de ausencia de valor. No es un sustituto de null en todos los casos, sino una herramienta para los tipos de retorno de metodos que pueden no tener un resultado.

## Creacion de Optional

Tres metodos factory estaticos cubren todos los escenarios de creacion:

```java
// Optional con valor no nulo (lanza NullPointerException si el argumento es null)
Optional<String> conValor = Optional.of("Hola");

// Optional que acepta null (devuelve Optional.empty() si el argumento es null)
Optional<String> puedeSerNulo = Optional.ofNullable(valor);

// Optional vacio
Optional<String> vacio = Optional.empty();
```

Usar `Optional.of(null)` lanza `NullPointerException` inmediatamente, lo cual es deseable cuando el contrato del metodo exige un valor no nulo. `Optional.ofNullable` es la opcion flexible para cuando no se sabe si el argumento es null.

## Comprobacion y extraccion

Los metodos clasicos para verificar presencia y obtener el valor deben usarse con conocimiento de sus implicaciones:

```java
isPresent()  // boolean: true si hay valor presente
isEmpty()    // boolean: true si esta vacio (Java 11+)
get()        // T: devuelve el valor o lanza NoSuchElementException
```

`get()` sin `isPresent()` previo es un antipatron. La API proporciona alternativas seguras que eliminan practicamente la necesidad de esta combinacion:

```java
Optional<String> opt = obtenerResultado();

if (opt.isPresent()) {
    String val = opt.get(); // aceptable solo con guarda
}
```

## orElse, orElseGet, orElseThrow

Estos metodos permiten extraer el valor con estrategias de fallback sin comprobacion manual:

```java
// Valor por defecto (siempre evaluado)
String resultado = opt.orElse("valor por defecto");

// Valor por defecto con Supplier (evaluacion perezosa)
String resultadoLazy = opt.orElseGet(() -> calcularDefault());

// Lanzar excepcion personalizada si ausente
String resultadoExigido = opt.orElseThrow(() ->
    new IllegalStateException("Valor requerido ausente"));

// Equivalente funcional a get() pero mas explicito
String resultadoSimple = opt.orElseThrow();
```

La diferencia entre `orElse` y `orElseGet` es relevante cuando el valor por defecto es costoso de calcular. `orElse` evalua el argumento incluso si el Optional tiene valor; `orElseGet` solo ejecuta el `Supplier` cuando el Optional esta vacio.

```java
// El metodo costoso() se ejecuta siempre
String r1 = opt.orElse(costoso());

// costoso() solo se ejecuta si opt esta vacio
String r2 = opt.orElseGet(() -> costoso());
```

## map, flatMap, filter

Estos metodos permiten transformar y filtrar el contenido de Optional sin extraerlo, siguiendo el mismo estilo funcional de los streams:

```java
// map: transforma el valor si presente
Optional<String> nombre = Optional.of("ana");
Optional<Integer> longitud = nombre.map(String::length);

// encadenamiento de map
String ciudad = Optional.of("madrid")
    .map(String::toUpperCase)
    .map(s -> "Capital: " + s)
    .orElse("Desconocida");
```

`flatMap` se usa cuando la funcion de transformacion devuelve otro `Optional`, evitando anidamientos:

```java
// Sin flatMap: Optional<Optional<Integer>>
Optional<Optional<Integer>> anidado = opt.map(s -> Optional.of(s.length()));

// Con flatMap: Optional<Integer>
Optional<Integer> plano = opt.flatMap(s -> Optional.of(s.length()));
```

`filter` evalua un predicado sobre el valor interno; si el predicado falla, devuelve `Optional.empty()`:

```java
Optional<String> largo = opt.filter(s -> s.length() > 5);
```

El encadenamiento completo permite escribir pipelines sin condicionales:

```java
String resultado = obtenerOptional()
    .filter(s -> s.startsWith("A"))
    .map(String::toUpperCase)
    .map(s -> s.substring(0, 3))
    .orElse("default");
```

## ifPresent e ifPresentOrElse

Cuando se necesita ejecutar una accion solo si hay valor presente:

```java
opt.ifPresent(valor -> System.out.println(valor));

// Java 9+: accion para presente y Runnable para ausente
opt.ifPresentOrElse(
    valor -> System.out.println("Valor: " + valor),
    () -> System.out.println("No hay valor")
);
```

## Uso en repositorios

El caso de uso canonico de `Optional` es el retorno de metodos de busqueda en repositorios o DAOs. En lugar de devolver null cuando no se encuentra una entidad, se devuelve `Optional<T>`:

```java
public interface UsuarioRepository {
    Optional<Usuario> findByEmail(String email);
    Optional<Usuario> findById(Long id);
    List<Usuario> findAll();
}
```

El consumidor del repositorio puede encadenar operaciones:

```java
Usuario usuario = usuarioRepository.findByEmail(email)
    .orElseThrow(() -> new UsuarioNoEncontradoException(email));

int longitudNombre = usuarioRepository.findById(id)
    .map(Usuario::getNombre)
    .map(String::length)
    .orElse(0);
```

> No debe usarse `Optional` para campos de entidades JPA, argumentos de metodos, o atributos de clases serializables. `Optional` no es serializable y su uso en esos contextos anade complejidad sin beneficio. Tampoco debe usarse como reemplazo de colecciones vacias: un `List` vacio es preferible a un `Optional<List>`.
