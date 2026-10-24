# Verificacion

Mockito no solo permite configurar comportamiento, sino tambien verificar que las interacciones esperadas ocurrieron con los argumentos y frecuencia adecuados. El metodo `verify()` es el punto de entrada para todas las comprobaciones de interaccion.

La forma mas simple verifica que un metodo fue invocado al menos una vez con los argumentos especificados:

```java
verify(repositorio).guardar(usuario);
```

Para controlar el numero exacto de invocaciones se usa el modificador `times()`:

```java
verify(repositorio, times(1)).guardar(usuario);   // exactamente una vez
verify(repositorio, times(3)).actualizar(usuario); // exactamente tres veces
```

`times(1)` es el valor por defecto, por lo que `verify(mock).metodo()` equivale a `verify(mock, times(1)).metodo()`.

Mockito proporciona modificadores adicionales para escenarios de verificacion comunes:

```java
verify(repositorio, never()).eliminar(anyLong());           // nunca invocado
verify(repositorio, atLeast(2)).buscar(anyString());         // al menos 2 veces
verify(repositorio, atLeastOnce()).buscar(anyString());      // al menos 1 vez
verify(repositorio, atMost(3)).buscar(anyString());          // como maximo 3 veces
verify(repositorio, only()).guardar(usuario);                // unica interaccion
```

El modificador `timeout()` permite verificar que una invocacion ocurre dentro de un periodo de tiempo determinado, util para pruebas con concurrencia:

```java
verify(repositorio, timeout(100)).guardar(usuario);
```

Timeout se combina con `times()` para verificar invocaciones multiples en ventanas temporales:

```java
verify(repositorio, timeout(200).times(2)).guardar(usuario);
```

> Timeout no es un sleep. Internamente, Mockito reintenta la verificacion hasta que se cumple el plazo o la condicion se satisface.

`verifyNoMoreInteractions()` verifica que no hubo interacciones adicionales con el mock mas alla de las ya verificadas:

```java
verify(repositorio).guardar(usuario);
verify(repositorio).enviarNotificacion(usuario);
verifyNoMoreInteractions(repositorio);
```

Si `repositorio` recibio cualquier otra llamada distinta a `guardar` o `enviarNotificacion`, la verificacion falla. `verifyNoMoreInteractions()` debe usarse con moderacion porque hace las pruebas fragiles: cualquier cambio en la implementacion que anada una llamada colateral rompe el test.

`verifyNoInteractions()` es mas estricto: verifica que el mock no recibio ninguna llamada en absoluto:

```java
verifyNoInteractions(repositorio);
```

Para verificar el orden de las invocaciones entre uno o varios mocks se usa `InOrder`:

```java
InOrder inOrder = inOrder(repositorio, servicioEmail);

// Verifica que guardar ocurre antes de enviar
inOrder.verify(repositorio).guardar(usuario);
inOrder.verify(servicioEmail).enviarBienvenida(usuario);
```

`InOrder` ejecuta verificaciones secuenciales. Si se invoca `enviarBienvenida` antes de `guardar`, la verificacion falla. Es posible mezclar multiples mocks en el mismo `InOrder` para validar protocolos de colaboracion complejos.

Mockito tambien permite ignorar verificaciones ya realizadas con `clearInvocations()` o `reset()`:

```java
// Limpia el registro de invocaciones sin perder los stubs
clearInvocations(repositorio);

// Resetea completamente el mock (stubs e invocaciones)
reset(repositorio);
```

> `reset()` es un indicador de que el test podria estar mal disenado. Es preferible crear un mock nuevo que resetear uno existente.

La verificacion fallida lanza `MockitoException` con un mensaje que describe que invocacion se esperaba, cuales fueron las llamadas reales, y cuantas veces ocurrio cada una. Mockito imprime las diferencias de forma legible para facilitar la depuracion, incluyendo los valores de los argumentos de cada invocacion.
