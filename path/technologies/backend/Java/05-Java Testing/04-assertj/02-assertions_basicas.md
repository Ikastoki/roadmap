# Assertions basicas

El corazon de AssertJ es el metodo estatico `assertThat`, que acepta cualquier objeto o primitiva y devuelve una instancia de la clase de asercion adecuada al tipo. Sobre esa instancia se encadenan los metodos de verificacion, conocidos como _assertion methods_. Las aserciones basicas cubren las operaciones fundamentales de comparacion, identidad, nulidad y booleanos.

## assertThat

`Assertions.assertThat(T actual)` es el unico punto de entrada necesario. Dependiendo del tipo del argumento, el compilador selecciona la sobrecarga correcta y retorna un asertor tipado:

```java
import static org.assertj.core.api.Assertions.assertThat;

assertThat("texto").isNotNull();
assertThat(42).isPositive();
assertThat(Arrays.asList(1, 2, 3)).hasSize(3);
```

No es necesario recordar nombres de metodos especificos para cada tipo; el IDE muestra las opciones disponibles gracias al tipado fuerte.

## isEqualTo / isNotEqualTo

Comparan el valor real contra un valor esperado usando `equals()`. Son el equivalente directo de `assertEquals` y `assertNotEquals` de JUnit, pero con mensajes de error mucho mas descriptivos.

```java
assertThat("Hola").isEqualTo("Hola");
assertThat(100).isNotEqualTo(200);
```

Cuando la comparacion falla, AssertJ muestra ambos valores y senala la diferencia:

```
expected: "Hola" but was: "hola"
```

> Para comparar arrays u objetos sin un `equals()` adecuado, AssertJ proporciona variantes como `isEqualToIgnoringCase` para strings o `usingRecursiveComparison` para objetos complejos.

## isNull / isNotNull

Verifican si el valor real es nulo o no. Son semanticamente equivalentes a `assertNull` y `assertNotNull` de JUnit.

```java
Object nulo = null;
assertThat(nulo).isNull();

Object noNulo = new Object();
assertThat(noNulo).isNotNull();
```

## isTrue / isFalse

Operan sobre valores booleanos o expresiones `Boolean`. Son mas legibles que `assertTrue` y `assertFalse` de JUnit porque nombran explicitamente la condicion esperada.

```java
boolean activo = true;
assertThat(activo).isTrue();
assertThat(usuario.isMayorDeEdad()).isFalse();
```

Tambien existe `isNotTrue` y `isNotFalse` para casos donde se quiere afirmar la negacion sin usar un `not()` explicito.

## isSameAs / isNotSameAs

Verifican la identidad de referencia mediante el operador `==`, no la igualdad logica via `equals()`. Son utiles cuando se necesita confirmar que dos variables apuntan exactamente al mismo objeto en memoria.

```java
List<String> lista = new ArrayList<>();
List<String> mismaReferencia = lista;
List<String> copia = new ArrayList<>(lista);

assertThat(lista).isSameAs(mismaReferencia);  // pasa: misma referencia
assertThat(lista).isNotSameAs(copia);          // pasa: distinto objeto
```

> La confusion entre `isEqualTo` e `isSameAs` es una de las causas mas frecuentes de falsos positivos en pruebas. `isEqualTo` usa `equals()` (logica de negocio), mientras que `isSameAs` usa `==` (identidad de memoria). Para objetos que no sobrescriben `equals()`, ambos metodos se comportan igual.

```java
assertThat("cadena").isEqualTo("cadena"); // equals()
assertThat("cadena").isSameAs("cadena");   // == (por String pool, puede coincidir)
```

La tabla siguiente resume las aserciones basicas:

| Metodo | Verifica |
|--------|----------|
| `isEqualTo(esperado)` | `actual.equals(esperado)` |
| `isNotEqualTo(esperado)` | `!actual.equals(esperado)` |
| `isNull()` | `actual == null` |
| `isNotNull()` | `actual != null` |
| `isTrue()` | `actual == true` |
| `isFalse()` | `actual == false` |
| `isSameAs(esperado)` | `actual == esperado` |
| `isNotSameAs(esperado)` | `actual != esperado` |

Todas estas aserciones devuelven el mismo asertor, lo que permite encadenar multiples verificaciones sobre un mismo valor sin almacenar variables intermedias.
