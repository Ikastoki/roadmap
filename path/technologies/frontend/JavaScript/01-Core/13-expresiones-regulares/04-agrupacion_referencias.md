<h1 align='center'>Agrupación y Referencias</h1>

<h2>📑 Contenido</h2>

- [Agrupación y Referencias](#agrupación-y-referencias)
- [Agrupación con Paréntesis ()](#agrupación-con-paréntesis-)
- [Captura de Subexpresiones (Grupos de Captura)](#captura-de-subexpresiones-grupos-de-captura)
- [Referencias hacia Atrás (Backreferences)](#referencias-hacia-atrás-backreferences)
- [Grupos No Capturantes (?:...)](#grupos-no-capturantes-)
- [Uso de Grupos en Reemplazos](#uso-de-grupos-en-reemplazos)
- [Referencias hacia Adelante (Lookahead y Lookbehind)](#referencias-hacia-adelante-lookahead-y-lookbehind)
  - [Lookahead Positivo (?=...)](#lookahead-positivo-)
  - [Lookahead Negativo (?!...)](#lookahead-negativo-)
  - [Lookbehind Positivo (?\<=...)](#lookbehind-positivo-)
  - [Lookbehind Negativo (?\<!...)](#lookbehind-negativo-)

## Agrupación y Referencias

Las expresiones regulares (RegExp) en JavaScript permiten no solo buscar patrones complejos en cadenas, sino también agrupar partes de dichos patrones y referirse a ellas más adelante. Esta capacidad es fundamental cuando queremos capturar partes específicas de una cadena para procesarlas o utilizarlas en reemplazos.

## Agrupación con Paréntesis ()

La agrupación en expresiones regulares se realiza usando paréntesis `()`. Los paréntesis permiten agrupar una porción del patrón de búsqueda, tratándola como una unidad. Esto es útil para aplicar cuantificadores a grupos enteros y para capturar subcadenas específicas para su posterior uso.

```js
let regex = /(abc)+/; // Coincide con una o más repeticiones de "abc"
console.log(regex.test("abcabc")); // true (hay dos repeticiones de "abc")
console.log(regex.test("abc")); // true (hay una repetición de "abc")
console.log(regex.test("ab")); // false (falta la "c" para coincidir con "abc")
```

En este ejemplo, el patrón `(abc)+` significa que la secuencia "abc" debe aparecer al menos una vez, y puede repetirse.

## Captura de Subexpresiones (Grupos de Captura)

Cuando usamos paréntesis `()`, las partes agrupadas del patrón también se capturan automáticamente. Estas subexpresiones capturadas (o grupos de captura) pueden ser referenciadas más adelante, lo cual es útil para obtener coincidencias específicas dentro de una cadena.

Cada grupo capturado se guarda en un índice basado en su orden de aparición, comenzando por el número `1`. El grupo `0` representa la coincidencia total.

```js
let regex = /(\d{4})-(\d{2})-(\d{2})/; // Captura una fecha en formato "YYYY-MM-DD"
let result = regex.exec("2024-09-20");

console.log(result[0]); // "2024-09-20" (la coincidencia completa)
console.log(result[1]); // "2024" (primer grupo capturado)
console.log(result[2]); // "09" (segundo grupo capturado)
console.log(result[3]); // "20" (tercer grupo capturado)
```

Aquí, el patrón `/(\d{4})-(\d{2})-(\d{2})/` captura una fecha en el formato "YYYY-MM-DD". Los tres grupos corresponden al año, mes y día.

## Referencias hacia Atrás (Backreferences)

Una referencia hacia atrás permite referirse a un grupo de captura dentro de la misma expresión regular. Esto es útil cuando queremos buscar patrones que se repitan.

Las referencias hacia atrás se realizan utilizando la barra invertida `\` seguida del número del grupo de captura al que se quiere referir. Por ejemplo, `\1` hace referencia al primer grupo capturado.

```js
let regex = /(\w)\1/; // Coincide con cualquier carácter alfanumérico que se repite dos veces seguidas
console.log(regex.test("aa")); // true (la "a" se repite dos veces)
console.log(regex.test("ab")); // false (los caracteres no son iguales)
```

En este ejemplo, el patrón `(\w)\1` busca una letra o dígito seguido de sí mismo, es decir, una repetición.

## Grupos No Capturantes (?:...)

A veces, es útil agrupar elementos para aplicarles un cuantificador, pero sin necesidad de capturar ese grupo para referenciarlo. Para esto se utilizan los grupos no capturantes, los cuales se crean usando `(?:...)`. Estos grupos no se almacenan ni se les asigna un número de referencia.

```js
let regex = /(?:abc)+/; // Coincide con una o más repeticiones de "abc" pero no captura
let result = regex.exec("abcabc");
console.log(result[0]); // "abcabc" (la coincidencia completa)
console.log(result[1]); // undefined (no hay grupo capturado)
```

En este caso, el patrón `(?:abc)+` busca una o más repeticiones de "abc", pero no captura la subexpresión en un grupo, por lo que `result[1]` es `undefined`.

## Uso de Grupos en Reemplazos

Los grupos capturados en una expresión regular también se pueden utilizar en operaciones de reemplazo mediante el método `replace()`. En los reemplazos, las referencias a los grupos capturados se hacen con el símbolo `$` seguido del número del grupo.

```js
let regex = /(\w+)\s(\w+)/; // Captura dos palabras separadas por un espacio
let str = "Hello World";
let newStr = str.replace(regex, "$2, $1"); // Invierte el orden de las palabras
console.log(newStr); // "World, Hello"
```

Aquí, la expresión regular `(\w+)\s(\w+)` captura dos palabras. Luego, el método `replace()` usa `$2` para referirse al segundo grupo (la segunda palabra) y `$1` para el primer grupo, intercambiándolos en la cadena resultante.

## Referencias hacia Adelante (Lookahead y Lookbehind)

Los lookahead y lookbehind permiten verificar si hay un patrón antes o después de un punto determinado sin capturarlo o consumirlo en la búsqueda.

### Lookahead Positivo (?=...)

Un lookahead positivo verifica que un patrón aparezca a continuación sin incluirlo en la coincidencia

```js
let regex = /\d(?=px)/; // Coincide con un dígito seguido de "px", pero no captura "px"
console.log(regex.exec("100px")); // ["0"] (solo el dígito se captura)
```

### Lookahead Negativo (?!...)

Un lookahead negativo asegura que un patrón no esté presente después de la coincidencia.

```js
let regex = /\d(?!px)/; // Coincide con un dígito que no esté seguido de "px"
console.log(regex.exec("100em")); // ["0"] (el "0" no está seguido de "px")
```

### Lookbehind Positivo (?<=...)

Un lookbehind positivo verifica que haya un patrón antes de la coincidencia.

```js
let regex = /(?<=\$)\d+/; // Coincide con un número que esté precedido por el símbolo "$"
console.log(regex.exec("$100")); // ["100"] (captura el número pero no el "$")
```

### Lookbehind Negativo (?<!...)

Un lookbehind negativo asegura que un patrón no esté presente antes de la coincidencia.

```js
let regex = /(?<!\$)\d+/; // Coincide con un número que no esté precedido por "$"
console.log(regex.exec("100")); // ["100"] (el número no está precedido por "$")
```
