# Spies

Un espia (`@Spy`) es un wrapper sobre una instancia real que permite observar y modificar selectivamente el comportamiento de sus metodos. Mientras que un mock reemplaza completamente el objeto, un espia mantiene la implementacion real y solo intercepta los metodos explicitamente stubeados.

La creacion de un espia se realiza con la anotacion `@Spy` o el metodo `spy()`:

```java
@Spy
private List<String> lista = new ArrayList<>();

// Equivalente funcional
List<String> lista = Mockito.spy(new ArrayList<>());
```

Cuando se invoca un metodo no stubeado en un espia, Mockito delega en la implementacion real del objeto. Esto contrasta con un mock, donde cualquier metodo no stubeado devuelve el valor por defecto del tipo de retorno:

```java
@Mock
private List<String> listaMock;        // listaMock.get(0) -> null

@Spy
private List<String> listaSpy = new ArrayList<>(); // listaSpy.get(0) -> IndexOutOfBoundsException (comportamiento real)
```

El stubbing parcial en espias permite reemplazar el comportamiento de metodos especificos mientras se mantiene el resto. Sin embargo, existe una diferencia crucial entre `when().thenReturn()` y `doReturn().when()` cuando se usan con espias:

```java
@Spy
private Procesador procesador = new Procesador();

// PROBLEMA: when() ejecuta el metodo real antes de configurar el stub
when(procesador.metodoCostoso()).thenReturn("mockeado");
// -> metodoCostoso() se ejecuta REALMENTE durante la configuracion

// SOLUCION: doReturn evita la ejecucion real
doReturn("mockeado").when(procesador).metodoCostoso();
```

Esta diferencia es el motivo principal por el que existe la sintaxis `doReturn().when()`. Cuando se usa `when()` con un espia, la expresion dentro de `when()` invoca el metodo real, lo que puede tener efectos secundarios no deseados si el metodo tiene side effects o es costoso. `doReturn()` configura el stub sin invocar el metodo.

> La regla practica es: usar `doReturn().when()` para espias, y `when().thenReturn()` para mocks puros. Aunque ambas sintaxis funcionan en ambos casos, solo `doReturn()` garantiza que no se ejecute el metodo real durante la configuracion del espia.

Los espias son adecuados cuando se prueba una clase concreta y se desea verificar su comportamiento interno o aislar una dependencia externa dentro de la misma clase. Ejemplos tipicos:

- Probar un metodo que llama internamente a otro metodo de la misma clase, stubeando el metodo interno.
- Verificar que un metodo interactua correctamente con sus propios metodos privados (aunque esto suele ser indicador de diseno mejorable).
- Probar clases legacy donde no es practico extraer interfaces.

La decision entre `@Spy` y `@Mock` se reduce a si se necesita comportamiento real por defecto o no. Un mock es preferible cuando se prueba la interaccion entre objetos y no importa la implementacion interna del colaborador. Un espia es preferible cuando se prueba una clase concreta y se quiere mantener su comportamiento salvo en puntos especificos.

Mockito permite resetear espias parcialmente con `reset()`:

```java
reset(espia);
// El espia pierde todos los stubs y vuelve al comportamiento real completo
```

Los espias tambien admiten verificacion exactamente igual que los mocks: `verify()` funciona de forma identica, contando invocaciones sobre metodos reales y stubeados.
