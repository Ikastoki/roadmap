# Stubbing (when/thenReturn)

El stubbing es el mecanismo mediante el cual se define el comportamiento de un mock ante llamadas especificas. Mockito proporciona una API fluida basada en `when()` y `thenReturn()` que permite configurar respuestas, excepciones y comportamientos personalizados.

La forma mas basica de stubbing usa `when()` para seleccionar el metodo y `thenReturn()` para definir la respuesta:

```java
when(repositorio.buscarPorId(1L)).thenReturn(usuarioEsperado);
```

Cuando se invoca `repositorio.buscarPorId(1L)` durante la prueba, Mockito intercepta la llamada y devuelve `usuarioEsperado` sin ejecutar la implementacion real. Cualquier otro argumento devuelve el valor por defecto del tipo de retorno (`null` en este caso).

Para lanzar excepciones controladamente se usa `thenThrow()`:

```java
when(repositorio.buscarPorId(-1L)).thenThrow(new IllegalArgumentException("ID invalido"));
```

Es posible encadenar multiples respuestas para llamadas sucesivas al mismo metodo con los mismos argumentos:

```java
when(iterador.next())
    .thenReturn("primero")
    .thenReturn("segundo")
    .thenThrow(new NoSuchElementException());
```

La primera llamada devuelve `"primero"`, la segunda `"segundo"`, y la tercera lanza la excepcion. Esta variante es equivalente a pasar los valores en secuencia:

```java
when(iterador.next()).thenReturn("primero", "segundo")
    .thenThrow(new NoSuchElementException());
```

`thenAnswer()` permite definir una respuesta que depende de los argumentos de la llamada o de estado externo. Recibe una implementacion de `Answer` cuyo metodo `answer(InvocationOnMock)` evalua en el momento de la invocacion:

```java
when(calculadora.sumar(anyInt(), anyInt()))
    .thenAnswer(invocation -> {
        int a = invocation.getArgument(0);
        int b = invocation.getArgument(1);
        return a + b;
    });
```

La diferencia fundamental entre `thenReturn` y `thenAnswer` es el momento de evaluacion. `thenReturn` evalua el argumento en el momento de configurar el stub; `thenAnswer` evalua en el momento de la invocacion. Esto es critico cuando el valor de retorno se construye dinamicamente:

```java
// MAL: la fecha se calcula al crear el stub
when(servicio.obtenerFecha()).thenReturn(new Date());

// BIEN: la fecha se calcula al invocar el metodo
when(servicio.obtenerFecha()).thenAnswer(invocation -> new Date());
```

Existe un estilo alternativo de stubbing con `doReturn()` y `doThrow()` que resulta util cuando el metodo que se desea stubear no puede usarse directamente con `when()`. Esto ocurre con metodos `void`, metodos `final`, metodos `static` (con mockito-inline), y especialmente con espias donde se quiere evitar ejecutar el metodo real durante la configuracion:

```java
// Stubbing de metodo void
doNothing().when(repositorio).eliminar(anyLong());

// Stubbing para evitar ejecucion real en spy
doReturn(true).when(espia).metodoLlamadoDuranteSetup();

// Lanzar excepcion en metodo void
doThrow(new RuntimeException("Error"))
    .when(servicio).ejecutar(anyString());
```

La regla general es: usar `when().thenReturn()` cuando sea posible, y `doReturn().when()` solo cuando el metodo no permita el primer estilo. El compilador de Java fuerza esta decision porque `when()` requiere que la expresion interna tenga un tipo de retorno no `void`.

> `doReturn().when()` no verifica en tiempo de compilacion que el tipo devuelto sea compatible con el metodo. Un error de tipo solo se detecta en ejecucion, a diferencia de `when().thenReturn()` que es type-safe.

Mockito permite stubear metodos para que lancen excepciones unchecked unicamente cuando se usa `when().thenThrow()`. Para excepciones checked, el metodo debe declararlas en su firma. `doThrow()` carece de esta restriccion porque no pasa por el sistema de tipos de la expresion interna.

El stubbing de propiedades con `thenReturn()` acepta tambien un varargs para configurar multiples respuestas en una sola linea, y la sobrecarga de `thenReturn()` con lista de valores permite un control granular sobre el comportamiento secuencial del mock.
