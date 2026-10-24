# Argument matchers

Los matchers permiten definir stubs o verificaciones sin conocer el valor exacto de los argumentos. Cuando un metodo debe comportarse igual para un rango de valores de entrada, los matchers evitan tener que enumerar cada caso individual.

El matcher mas generico es `any()`, que acepta cualquier valor, incluyendo `null`. Su variante tipada `any(Class<T>)` ofrece seguridad de tipos:

```java
when(repositorio.buscarPorId(anyLong())).thenReturn(usuarioGenerico);
when(servicio.procesar(any(Usuario.class))).thenReturn(resultado);
```

Para valores exactos se usa `eq()`, que compara con `equals()`:

```java
when(repositorio.buscarPorNombre(eq("admin"))).thenReturn(usuarioAdmin);
```

`eq()` es necesario cuando se mezclan matchers con valores literales. Si un argumento usa matcher, todos los argumentos de la misma llamada deben usar matchers:

```java
// CORRECTO
when(repositorio.actualizar(eq(1L), any(Usuario.class))).thenReturn(true);

// INCORRECTO: no se puede mezclar literal con matcher
// when(repositorio.actualizar(1L, any(Usuario.class))).thenReturn(true);
```

Mockito proporciona matchers para cadenas de texto:

```java
when(servicio.buscar(contains("error"))).thenReturn(listaErrores);
when(servicio.buscar(startsWith("tmp_"))).thenReturn(listaTemporales);
when(servicio.buscar(endsWith(".log"))).thenReturn(listaLogs);
when(servicio.buscar(matches("\\d{4}-.*"))).thenReturn(listaFiltrada);
```

Para colecciones y mapas existen matchers especificos:

```java
when(repositorio.buscarTodos(anyCollection())).thenReturn(lista);
when(repositorio.buscarPorIds(anySet())).thenReturn(lista);
when(repositorio.buscarPorFiltro(anyMap())).thenReturn(lista);
```

El matcher `isA()` verifica el tipo del argumento en tiempo de ejecucion, mientras que `any()` no realiza ninguna comprobacion:

```java
when(servicio.procesar(isA(Usuario.class))).thenReturn(true); // solo acepta Usuario o subclases
```

Para casos donde los matchers predefinidos no cubren la necesidad, `argThat()` permite proporcionar una expresion lambda que implementa la logica de匹配:

```java
when(repositorio.buscar(argThat(arg -> arg != null && arg.getId() > 0)))
    .thenReturn(usuarioValido);
```

La diferencia fundamental entre matchers y valores exactos radica en el momento de evaluacion y la intencion semantica. Usar un valor exacto indica que el test conoce el argumento preciso que debe recibir el metodo. Usar un matcher indica que el argumento puede variar, pero debe cumplir cierta condicion. Elegir uno u otro afecta la legibilidad y el mantenimiento: los valores exactos hacen el test mas especifico pero mas fragil; los matchers lo hacen mas flexible pero potencialmente mas permisivo.

> Cuando un matcher es demasiado permisivo, puede ocultar errores. Por ejemplo, `anyString()` aceptaria un `null` silenciosamente si el metodo no espera nulos. Preferir matchers especificos como `isA(String.class)` cuando se quiera excluir `null`.

Es posible usar matchers en verificaciones:

```java
verify(repositorio).guardar(argThat(u -> u.getEmail().contains("@")));
```

La mezcla de matchers y valores literales sigue la misma regla que en stubbing: si se usa un matcher para un argumento, deben usarse matchers para todos los argumentos de ese metodo.

```java
// Correcto
verify(repositorio).actualizar(eq(1L), any(Usuario.class));

// Incorrecto
// verify(repositorio).actualizar(1L, any(Usuario.class));
```

Los matchers de Mockito son objetos de tipo `Matcher` que se registran en un stack interno. Por eso es importante no usar matchers fuera del contexto de `when()` o `verify()`, ya que quedarian en el stack y provocarian `InvalidUseOfMatchersException`.
