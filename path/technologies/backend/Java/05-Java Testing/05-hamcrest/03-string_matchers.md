# String matchers

Hamcrest proporciona un conjunto de matchers especificos para cadenas de texto, definidos en `org.hamcrest.Matchers` y tambien en la clase dedicada `org.hamcrest.text.StringContainsInOrder`, `IsEmptyString`, `IsBlankString`, entre otras. Estos matchers permiten verificar contenido, prefijos, sufijos, patrones regulares y estado de la cadena (vacia o en blanco) de forma declarativa.

## containsString / containsStringIgnoringCase

`containsString` verifica que la cadena real contenga la subcadena especificada. La busqueda es sensible a mayusculas. `containsStringIgnoringCase` (disponible en Hamcrest 2.x) realiza la misma busqueda ignorando diferencias de capitalizacion.

```java
String mensaje = "Error 500: Internal Server Error";

assertThat(mensaje, containsString("500"));
assertThat(mensaje, containsString("Internal"));
assertThat(mensaje, containsStringIgnoringCase("internal"));
assertThat(mensaje, not(containsString("200")));
```

A diferencia de `String.contains()`, el matcher produce mensajes de error descriptivos que muestran tanto la cadena original como la subcadena buscada.

## startsWith / startsWithIgnoringCase

`startsWith` verifica que la cadena comience con el prefijo especificado. Al igual que `containsString`, existe la variante que ignora mayusculas.

```java
String ruta = "/api/v2/usuarios";

assertThat(ruta, startsWith("/api"));
assertThat(ruta, startsWithIgnoringCase("/API"));
assertThat(ruta, not(startsWith("/v1")));
```

## endsWith / endsWithIgnoringCase

`endsWith` verifica el sufijo de la cadena. Es funcionalmente equivalente a `String.endsWith()` pero integrado en el ecosistema Hamcrest.

```java
String archivo = "documento.pdf";

assertThat(archivo, endsWith(".pdf"));
assertThat(archivo, endsWithIgnoringCase(".PDF"));
assertThat(archivo, not(endsWith(".txt")));
```

## matchesRegex / matchesPattern

`matchesRegex` (o `matchesPattern` en versiones recientes) verifica que la cadena completa coincida con una expresion regular. La regex se pasa como `String` o como `Pattern` compilado.

```java
String email = "usuario@dominio.com";

assertThat(email, matchesRegex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"));
assertThat("123-456-7890", matchesRegex("\\d{3}-\\d{3}-\\d{4}"));
```

> La diferencia entre `matchesRegex` y `containsString` es la misma que entre `String.matches()` y `String.contains()`: `matchesRegex` exige que la regex case con la cadena completa, mientras que `containsString` solo busca una subcadena literal. Para buscar un patron dentro de la cadena sin exigir coincidencia total, puede combinarse `containsString` con una porcion fija o usar directamente un matcher personalizado.

```java
// Coincidencia parcial usando find()
import static org.hamcrest.text.MatchesPattern.matchesPattern;

assertThat("codigo ERROR-42 en linea 10", matchesPattern(".*ERROR-\\d+.*"));
```

## emptyString / emptyOrNullString / blankString / blankOrNullString

Estos matchers verifican el estado de la cadena respecto a su contenido. `emptyString` comprueba que `length() == 0`. `emptyOrNullString` ademas acepta `null` como valido. `blankString` verifica que la cadena solo contenga caracteres blancos (espacios, tabs, saltos de linea). `blankOrNullString` extiende la verificacion para aceptar `null`.

```java
assertThat("", emptyString());
assertThat("   ", blankString());
assertThat((String) null, emptyOrNullString());
assertThat((String) null, blankOrNullString());
assertThat("Hola", not(emptyString()));
```

> Estos matchers se encuentran en `org.hamcrest.text.IsEmptyString` y `org.hamcrest.text.IsBlankString`. Si se importa `org.hamcrest.Matchers.*`, estan disponibles como `emptyString()`, `isEmptyString()`, `isEmptyOrNullString()`, `blankString()` y `blankOrNullString()`.

```java
// Uso combinado
String valor = "  ";
assertThat(valor, anyOf(blankString(), emptyString()));
assertThat(valor, both(not(emptyString())).and(blankString()));
```

Hamcrest proporciona ademas `stringContainsInOrder`, que verifica que una cadena contenga varias subcadenas en el orden especificado (no necesariamente consecutivas):

```java
assertThat("La lista de espera esta completa",
    stringContainsInOrder(Arrays.asList("lista", "espera", "completa")));
```
