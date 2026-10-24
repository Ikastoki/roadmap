# Crear mocks

Mockito ofrece dos mecanismos principales para crear mocks: el metodo estatico `Mockito.mock()` y las anotaciones `@Mock` y `@Spy`. Ambos enfoques son equivalentes en funcionalidad; la eleccion depende del estilo y la claridad del codigo de prueba.

`Mockito.mock()` recibe la clase a mockear y devuelve un proxy que implementa todos sus metodos con valores por defecto. La forma mas simple:

```java
List<String> listaMock = Mockito.mock(List.class);
```

El mock creado responde a cualquier llamada con el valor neutro del tipo de retorno: `null` para objetos, `0` para primitivas numericas, `false` para booleanos, y colecciones vacias para `Collection`/`Map`. Se puede refinar este comportamiento proporcionando un `Answer` por defecto via `MockSettings`:

```java
List<String> listaMock = Mockito.mock(List.class, Mockito.RETURNS_SMART_NULLS);
```

`RETURNS_SMART_NULLS` lanza excepciones descriptivas si el codigo intenta usar un valor no stubeado, en lugar de devolver `null` silenciosamente.

La anotacion `@Mock` logra el mismo resultado de forma declarativa, reduciendo ruido en el codigo:

```java
@Mock
private RepositorioUsuario repositorio;
```

`@Spy` crea un espia sobre una instancia real. A diferencia de un mock puro, el espia ejecuta el codigo real del metodo a menos que se stubee explicitamente. Se puede crear con una instancia existente o dejando que Mockito la instancie mediante un constructor sin argumentos:

```java
@Spy
private List<String> lista = new ArrayList<>();

// o delegando la creacion a Mockito
@Spy
private ProcesadorDatos procesador;
```

Los mocks parciales combinan comportamiento real con stubbing selectivo. Se construyen con `Mockito.mock()` pasando `Mockito.CALLS_REAL_METHODS` como respuesta por defecto:

```java
Calculadora calc = Mockito.mock(Calculadora.class, Mockito.CALLS_REAL_METHODS);
```

En este modo, los metodos no stubeados ejecutan la implementacion real, mientras que los stubeados devuelven lo configurado. Es util para aislar un metodo concreto de una clase que tiene dependencias internas complejas.

> El uso excesivo de `CALLS_REAL_METHODS` puede ocultar problemas de diseno. Si se necesita con frecuencia, la clase probablemente viola el principio de responsabilidad unica.

Mockito maneja tipos genericos correctamente siempre que la informacion de tipo este disponible en tiempo de compilacion. Para clases con parametros de tipo, el metodo `mock()` preserva la informacion generica si se usa la sintaxis estandar:

```java
// Correcto: el tipo generico se infiere
@Mock
private Repositorio<Usuario> repositorio;

// Equivalente funcional
private Repositorio<Usuario> repositorio = Mockito.mock(Repositorio.class);
```

Sin embargo, `Mockito.mock()` por si solo pierde el tipo generico en tiempo de ejecucion debido al borrado de tipos de Java. Las anotaciones con `@Mock` no sufren este problema porque el compilador incorpora la informacion generica en el bytecode del campo. Para casos donde se necesita el tipo generico en tiempo de ejecucion (por ejemplo, con `ArgumentCaptor`), es necesario proporcionar el tipo explicitamente mediante `Mockito.mock(Class)` con la clase raw.

La creacion de mocks para clases finales o metodos estaticos requiere el artefacto `mockito-inline` en el classpath. Desde Mockito 4.x, esta funcionalidad esta disponible de forma nativa en `mockito-core` pero deshabilitada por defecto; se activa con `mockito-inline` o configurando el archivo `org.mockito.plugins.MockMaker`.
