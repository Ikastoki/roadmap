# Strings

AssertJ proporciona un conjunto completo de aserciones especificas para cadenas de texto a traves de la clase `AbstractStringAssert`. Estas aserciones cubren desde verificaciones de contenido y tamaño hasta patrones regulares y normalizacion Unicode, permitiendo expresar condiciones complejas sobre strings de forma legible y encadenable.

## isEmpty / isNotEmpty / isBlank / isNotBlank

Verifican si una cadena esta vacia, tiene contenido, o solo contiene caracteres blancos. La diferencia entre `isEmpty` y `isBlank` es sutil pero importante: `isEmpty` comprueba que `length() == 0`, mientras que `isBlank` tambien acepta espacios, tabulaciones y saltos de linea.

```java
assertThat("").isEmpty();
assertThat("  ").isBlank();
assertThat("Hola").isNotEmpty();
assertThat("  ").isNotBlank(); // falla, es blank
```

## contains / doesNotContain

Verifican si la cadena contiene una o varias subcadenas. La sobrecarga que acepta varargs permite comprobar multiples ocurrencias en una sola llamada.

```java
String mensaje = "Error: usuario no encontrado";

assertThat(mensaje).contains("usuario");
assertThat(mensaje).contains("Error", "encontrado");
assertThat(mensaje).doesNotContain("permiso");
```

> `contains` busca subcadenas literales, no patrones. Para expresiones regulares debe usarse `matches`. Tampoco verifica orden; para eso existe `containsSequence`.

## startsWith / endsWith

Verifican el prefijo y sufijo de la cadena, equivalentes a `String.startsWith()` y `String.endsWith()` pero con la ventaja de integrarse en la cadena fluida.

```java
String ruta = "/api/usuarios/123";

assertThat(ruta).startsWith("/api");
assertThat(ruta).endsWith("123");
assertThat(ruta).startsWith("/api").endsWith("123");
```

## matches / doesNotMatch

Evaluan la cadena completa contra una expresion regular. El patron se proporciona como `String` o como `Pattern` compilado. La asercion `matches` exige que la regex case con la totalidad de la cadena, no con una subcadena.

```java
String email = "usuario@example.com";

assertThat(email).matches("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
assertThat(email).doesNotMatch("^[0-9]+$");
```

Para buscar un patron dentro de la cadena sin exigir coincidencia total, debe combinarse con `containsPattern`:

```java
assertThat("codigo: ERROR-42").containsPattern("ERROR-\\d+");
```

## hasSize / hasLineCount / hasSameSizeAs

`hasSize` verifica la longitud de la cadena mediante `length()`. `hasLineCount` cuenta los saltos de linea para determinar el numero de lineas. `hasSameSizeAs` compara la longitud contra otro objeto CharSequence.

```java
String texto = "linea1\nlinea2\nlinea3";

assertThat("abc").hasSize(3);
assertThat(texto).hasLineCount(3);
assertThat("abc").hasSameSizeAs("xyz");
```

## Otras aserciones utiles

AssertJ incluye decenas de metodos adicionales para strings: `isLowerCase` / `isUpperCase`, `hasDigits`, `hasOnlyDigits`, `isSubstringOf`, `hasSameSizeAs`, `containsOnlyDigits`, `containsWhitespaces`, `normalize` (para normalizacion Unicode), y `isXmlEqualTo` / `isJsonEqualTo` para comparacion estructurada de formatos.

```java
assertThat("ABC").isUpperCase();
assertThat("123").containsOnlyDigits();
assertThat("<tag>contenido</tag>").isXmlEqualTo("<tag>contenido</tag>");
```

> Las aserciones de XML y JSON realizan una comparacion semantica, no literal. Dos documentos XML con diferentes espacios en blanco o atributos en distinto orden se consideran iguales si su contenido semantico coincide. Esto las hace muy utiles para probar serializadores y respuestas de APIs.
