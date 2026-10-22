<h1 align='center'>Anclas y Limites</h1>

<h2>📑 Contenido</h2>

- [Anclas y Limites](#anclas-y-limites)
- [¿Qué son las Anclas y Límites en Expresiones Regulares?](#qué-son-las-anclas-y-límites-en-expresiones-regulares)
- [Ancla ^: Inicio de la Cadena o Línea](#ancla--inicio-de-la-cadena-o-línea)
  - [Ancla en múltiples líneas (con la bandera m)](#ancla-en-múltiples-líneas-con-la-bandera-m)
- [Ancla $: Final de la Cadena o Línea](#ancla--final-de-la-cadena-o-línea)
  - [Ancla en múltiples líneas (con la bandera m)](#ancla-en-múltiples-líneas-con-la-bandera-m-1)
- [Límite de Palabra \\b: Frontera entre Palabras](#límite-de-palabra-b-frontera-entre-palabras)
- [No Límite de Palabra \\B: No es Frontera de Palabra](#no-límite-de-palabra-b-no-es-frontera-de-palabra)
- [Combinación de Anclas y Límites](#combinación-de-anclas-y-límites)
- [Aplicaciones Prácticas](#aplicaciones-prácticas)

## Anclas y Limites

Las anclas y límites en las expresiones regulares (RegExp) son herramientas clave que permiten especificar posiciones precisas dentro de una cadena de texto, en lugar de caracteres específicos. En lugar de coincidir con un carácter, como las letras o números, las anclas y límites indican ubicaciones, como el comienzo o el final de una cadena, o las transiciones entre diferentes tipos de caracteres (alfanuméricos y no alfanuméricos).

Este capítulo explora las anclas más comunes, cómo funcionan y cómo puedes utilizarlas para crear patrones más precisos y específicos en tus búsquedas.

## ¿Qué son las Anclas y Límites en Expresiones Regulares?

En una expresión regular, una ancla se utiliza para marcar una posición en la cadena, en lugar de un carácter. Las anclas más comunes son `^` (que indica el inicio de una cadena o línea) y `$` (que indica el final de una cadena o línea). Un límite es una posición entre dos caracteres que define la transición de un tipo de carácter a otro, como el límite entre una letra y un espacio.

Las anclas no consumen caracteres de la cadena, simplemente indican una ubicación en la misma.

## Ancla ^: Inicio de la Cadena o Línea

El símbolo `^` se usa como una ancla que coincide con el inicio de una cadena o una línea (cuando se usa en conjunto con la bandera `m` para multilíneas). Este símbolo indica que el patrón debe empezar en la primera posición de la cadena para que haya una coincidencia.

```js
let regex = /^Hola/;
console.log(regex.test("Hola mundo")); // true (comienza con "Hola")
console.log(regex.test("Mundo, Hola")); // false (no comienza con "Hola")
```

En este ejemplo, la expresión `^Hola` solo coincide si la cadena comienza con "Hola". Cualquier otra palabra al principio de la cadena no dará una coincidencia.

### Ancla en múltiples líneas (con la bandera m)

```js
let regex = /^Hola/m;
let texto = `Adiós mundo
Hola de nuevo`;
console.log(texto.match(regex)); // ["Hola"]
```

Con la bandera `m`, la ancla `^` no solo marca el inicio de la cadena, sino también el inicio de cada nueva línea.

## Ancla $: Final de la Cadena o Línea

El símbolo `$` se usa para coincidir con el final de una cadena o línea. Esto significa que el patrón que precede a `$` debe coincidir justo antes de que la cadena termine, o antes de un salto de línea si se usa con la bandera m.

```js
let regex = /mundo$/;
console.log(regex.test("Hola mundo")); // true (termina con "mundo")
console.log(regex.test("mundo, Hola")); // false (no termina con "mundo")
```

Aquí, el patrón `mundo$` solo coincide si "mundo" está al final de la cadena.

### Ancla en múltiples líneas (con la bandera m)

```js
let regex = /adiós$/m;
let texto = `Hola de nuevo
Nos vemos, adiós`;
console.log(texto.match(regex)); // ["adiós"]
```

Con la bandera `m`, el símbolo `$` marca el final de cada línea, permitiendo coincidir con el texto antes del salto de línea.

## Límite de Palabra \b: Frontera entre Palabras

El límite de palabra `\b` es una posición entre un carácter alfanumérico (`\w`, que incluye letras, números y guiones bajos) y un carácter no alfanumérico (espacios, puntuación, etc.). Es útil para asegurarse de que un patrón solo coincida con palabras completas y no con partes de palabras.

```js
let regex = /\bperro\b/;
console.log(regex.test("El perro está aquí.")); // true (coincide con la palabra completa "perro")
console.log(regex.test("perrocaliente")); // false (es parte de una palabra más larga)
```

En este ejemplo, el patrón `\bperro\b` asegura que "perro" sea una palabra completa, no una subcadena dentro de otra palabra como "perrocaliente".

## No Límite de Palabra \B: No es Frontera de Palabra

El no límite de palabra `\B` indica cualquier posición que no sea un límite de palabra. Esto significa que coincide dentro de una palabra o en cualquier lugar donde no haya una transición entre caracteres alfanuméricos y no alfanuméricos.

```js
let regex = /\Bperro\B/;
console.log(regex.test("superperro")); // true (coincide con "perro" dentro de "superperro")
console.log(regex.test("perro")); // false (es una palabra completa)
```

En este caso, el patrón `\Bperro\B` busca la cadena "perro" solo cuando está dentro de otra palabra, como "superperro", y no cuando es una palabra aislada.

## Combinación de Anclas y Límites

Las anclas y los límites se pueden combinar para crear patrones más específicos y controlar con precisión las posiciones dentro de una cadena de texto.

```js
let regex = /^\bHola\b/;
console.log(regex.test("Hola mundo")); // true (comienza con la palabra "Hola")
console.log(regex.test("Holaamigo")); // false (no es una palabra completa)
console.log(regex.test("Adiós Hola")); // false (no está al inicio de la cadena)
```

Aquí, la expresión `^\bHola\b` asegura que "Hola" sea la primera palabra completa en la cadena.

## Aplicaciones Prácticas

Las anclas y límites son útiles en una variedad de situaciones de búsqueda y manipulación de texto:

- **Validación de formularios:** Verificar si un número de teléfono o código postal está completo desde el principio o hasta el final de la cadena.
- **Extracción de datos:** Extraer palabras completas en lugar de subcadenas, como nombres o palabras clave.
- **Filtrado de texto:** Encontrar solo palabras completas o eliminar caracteres innecesarios en los bordes de cadenas de texto.

```js
// Validar si una cadena es un número entero positivo
let regex = /^\d+$/;
console.log(regex.test("12345")); // true (solo contiene dígitos)
console.log(regex.test("12345a")); // false (contiene letras)
console.log(regex.test(" 12345 ")); // false (contiene espacios)
```

La expresión regular` ^\d+$` asegura que la cadena solo contenga dígitos (`\d+`), desde el principio (`^`) hasta el final (`$`), lo que es útil para validar números enteros.
