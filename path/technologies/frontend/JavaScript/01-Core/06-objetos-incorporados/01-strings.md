<h1 align='center'>Strings</h1>

<h2>📑 Contenido</h2>

- [Strings](#strings)
- [Propiedades de String](#propiedades-de-string)
  - [length](#length)
- [Métodos de Búsqueda y Comparación](#métodos-de-búsqueda-y-comparación)
  - [charAt()](#charat)
  - [charCodeAt()](#charcodeat)
  - [includes()](#includes)
  - [indexOf()](#indexof)
  - [lastIndexOf()](#lastindexof)
  - [startsWith()](#startswith)
  - [endsWith()](#endswith)
  - [localeCompare()](#localecompare)
- [Métodos de Manipulación](#métodos-de-manipulación)
  - [concat()](#concat)
  - [slice()](#slice)
  - [substring()](#substring)
  - [substr()](#substr)
  - [replace()](#replace)
  - [replaceAll()](#replaceall)
  - [split()](#split)
- [Métodos de Transformación](#métodos-de-transformación)
  - [toUpperCase()](#touppercase)
  - [toLowerCase()](#tolowercase)
  - [trim()](#trim)
  - [trimStart() / trimLeft()](#trimstart--trimleft)
  - [trimEnd() / trimRight()](#trimend--trimright)
  - [padStart()](#padstart)
  - [padEnd()](#padend)
  - [repeat()](#repeat)
- [Métodos de Conversión](#métodos-de-conversión)
  - [toString()](#tostring)
  - [valueOf()](#valueof)
- [Métodos de Patrones (Expresiones Regulares)](#métodos-de-patrones-expresiones-regulares)
  - [match()](#match)
  - [matchAll()](#matchall)
  - [search()](#search)
  - [replace()](#replace-1)
  - [replaceAll()](#replaceall-1)

## Strings

En JavaScript, las cadenas de texto (o strings) son uno de los tipos de datos primitivos más importantes. El objeto String proporciona un conjunto completo de propiedades y métodos para la búsqueda, manipulación, comparación y transformación de cadenas de texto. Este capítulo cubre de manera detallada las propiedades y métodos que nos ofrece este objeto incorporado.

## Propiedades de String

### length

La propiedad length devuelve el número de caracteres en una cadena de texto, incluyendo espacios y símbolos. Es útil para saber la longitud de una cadena antes de manipularla.

```js
const mensaje = "Hola, mundo";
console.log(mensaje.length); // Imprime: 10
```

## Métodos de Búsqueda y Comparación

### charAt()

Este método devuelve el carácter que se encuentra en una posición específica de la cadena. El índice se basa en cero, lo que significa que el primer carácter tiene índice `0`.

```js
const texto = "JavaScript";
console.log(texto.charAt(0)); // Imprime: "J"
```

### charCodeAt()

Devuelve el valor Unicode (o código ASCII) del carácter en una posición específica.

```js
const letra = "A";
console.log(letra.charCodeAt(0)); // Imprime: 65
```

### includes()

Este método comprueba si una subcadena está contenida dentro de la cadena original. Devuelve `true` si encuentra la subcadena y `false` en caso contrario.

```js
const frase = "Aprender JavaScript es divertido";
console.log(frase.includes("JavaScript")); // Imprime: true
console.log(frase.includes("Python")); // Imprime: false
```

### indexOf()

Busca la primera aparición de un valor específico en la cadena y devuelve su posición. Si no lo encuentra, devuelve `-1`.

```js
const frase = "Aprender JavaScript";
console.log(frase.indexOf("JavaScript")); // Imprime: 9
console.log(frase.indexOf("Python")); // Imprime: -1
```

### lastIndexOf()

Similar a `indexOf()`, pero busca la última aparición del valor en la cadena.

```js
const frase = "JavaScript, JavaScript y más JavaScript";
console.log(frase.lastIndexOf("JavaScript")); // Imprime: 31
```

### startsWith()

Verifica si una cadena comienza con una subcadena específica. Devuelve `true` o `false`.

```js
const saludo = "Hola, mundo";
console.log(saludo.startsWith("Hola")); // Imprime: true
```

### endsWith()

Verifica si una cadena termina con una subcadena específica.

```js
const saludo = "Hola, mundo";
console.log(saludo.endsWith("mundo")); // Imprime: true
```

### localeCompare()

Compara dos cadenas en orden lexicográfico, basado en reglas de idioma específicas, devolviendo `-1`, `0` o `1` dependiendo del resultado de la comparación.

```js
const cadena1 = "a";
const cadena2 = "b";
console.log(cadena1.localeCompare(cadena2)); // Imprime: -1 (porque "a" es menor que "b")
```

## Métodos de Manipulación

### concat()

Este método concatena (une) una o más cadenas y devuelve una nueva cadena resultante.

```js
const saludo = "Hola";
const nombre = "Ana";
console.log(saludo.concat(", ", nombre)); // Imprime: "Hola, Ana"
```

### slice()

Extrae una parte de la cadena, definida por índices de inicio y fin. El índice de fin es opcional y no se incluye en la nueva cadena.

```js
const texto = "JavaScript es increíble";
console.log(texto.slice(0, 10)); // Imprime: "JavaScript"
```

### substring()

Similar a `slice()`, pero no acepta valores negativos para los índices.

```js
const texto = "JavaScript";
console.log(texto.substring(0, 4)); // Imprime: "Java"
```

### substr()

Extrae una parte de la cadena, tomando como parámetros un índice de inicio y un número de caracteres a extraer. (Nota: `substr()` esta obsoleto y no se recomienda para código moderno).

```js
const texto = "JavaScript";
console.log(texto.substring(0, 4)); // Imprime: "Java"
```

### replace()

Busca un valor en la cadena y lo reemplaza con otro valor.

```js
const frase = "JavaScript es difícil";
const nuevaFrase = frase.replace("difícil", "fácil");
console.log(nuevaFrase); // Imprime: "JavaScript es fácil"
```

### replaceAll()

Reemplaza todas las ocurrencias de un valor en la cadena.

```js
const frase = "JavaScript, JavaScript y más JavaScript";
const nuevaFrase = frase.replaceAll("JavaScript", "JS");
console.log(nuevaFrase); // Imprime: "JS, JS y más JS"
```

### split()

Divide una cadena en un array de subcadenas, usando un delimitador.

```js
const listaFrutas = "manzana,pera,plátano";
const frutas = listaFrutas.split(",");
console.log(frutas); // Imprime: ["manzana", "pera", "plátano"]
```

## Métodos de Transformación

### toUpperCase()

Convierte todos los caracteres de una cadena a mayúsculas.

```js
const texto = "hola";
console.log(texto.toUpperCase()); // Imprime: "HOLA"
```

### toLowerCase()

Convierte todos los caracteres de una cadena a minúsculas.

```js
const texto = "HOLA";
console.log(texto.toLowerCase()); // Imprime: "hola"
```

### trim()

Elimina los espacios en blanco al principio y al final de la cadena.

```js
const mensaje = "   Hola, mundo   ";
console.log(mensaje.trim()); // Imprime: "Hola, mundo"
```

### trimStart() / trimLeft()

Elimina los espacios en blanco solo al inicio de la cadena.

```js
const mensaje = "   Hola, mundo";
console.log(mensaje.trimStart()); // Imprime: "Hola, mundo"
```

### trimEnd() / trimRight()

Elimina los espacios en blanco solo al final de la cadena.

```js
const mensaje = "   Hola, mundo";
console.log(mensaje.trimStart()); // Imprime: "Hola, mundo"
```

### padStart()

Rellena la cadena desde el inicio con un valor especificado hasta alcanzar la longitud deseada.

```js
const numero = "5";
console.log(numero.padStart(3, "0")); // Imprime: "005"
```

### padEnd()

Rellena la cadena desde el final con un valor especificado hasta alcanzar la longitud deseada.

```js
const numero = "5";
console.log(numero.padEnd(3, "0")); // Imprime: "500"
```

### repeat()

Devuelve una nueva cadena que repite la cadena original un número específico de veces.

```js
const texto = "ha";
console.log(texto.repeat(3)); // Imprime: "hahaha"
```

## Métodos de Conversión

### toString()

Devuelve una representación de la cadena como texto (comúnmente implícito cuando usamos una cadena en contextos donde se necesita una cadena de texto).

```js
const numero = 123;
console.log(numero.toString()); // Imprime: "123"
```

### valueOf()

Devuelve el valor primitivo de un objeto `String`. Normalmente no se utiliza explícitamente ya que JavaScript lo hace automáticamente.

```js
const cadena = new String("Hola");
console.log(cadena.valueOf()); // Imprime: "Hola"
```

## Métodos de Patrones (Expresiones Regulares)

### match()

Busca coincidencias de un patrón (expresión regular) en la cadena y devuelve un array con los resultados.

```js
const texto = "El color es rojo, no azul.";
const coincidencias = texto.match(/rojo/);
console.log(coincidencias); // Imprime: ["rojo"]
```

### matchAll()

Devuelve un iterador de todas las coincidencias de una expresión regular.

```js
const texto = "1a 2b 3c";
const regex = /\d\w/g;
for (const coincidencia of texto.matchAll(regex)) {
  console.log(coincidencia);
}
// Imprime:
// ["1a"]
// ["2b"]
// ["3c"]
```

### search()

Busca la primera coincidencia de una expresión regular y devuelve el índice donde ocurre.

```js
const texto = "El cielo es azul";
console.log(texto.search(/azul/)); // Imprime: 13
```

### replace()

Reemplaza coincidencias de una expresión regular con un nuevo valor.

```js
const texto = "El color es rojo";
const nuevoTexto = texto.replace(/rojo/, "azul");
console.log(nuevoTexto); // Imprime: "El color es azul"
```

### replaceAll()

Reemplaza todas las coincidencias de un patrón con un nuevo valor.

```js
const texto = "rojo rojo rojo";
const nuevoTexto = texto.replaceAll(/rojo/g, "azul");
console.log(nuevoTexto); // Imprime: "azul azul azul"
```
