<h1 align='center'>Caracteres</h1>

<h2>📑 Contenido</h2>

- [Caracteres](#caracteres)
- [Clases de Caracteres (`[...]`)](#clases-de-caracteres-)
  - [Ejemplo con rangos de caracteres](#ejemplo-con-rangos-de-caracteres)
- [Negación de Clases de Caracteres (`[^...]`)](#negación-de-clases-de-caracteres-)
  - [Ejemplo con rango de caracteres negados](#ejemplo-con-rango-de-caracteres-negados)
- [Clases Predefinidas](#clases-predefinidas)
  - [`\d` (Dígitos)](#d-dígitos)
  - [`\D` (No dígitos)](#d-no-dígitos)
  - [`\w` (Caracteres alfanuméricos)](#w-caracteres-alfanuméricos)
  - [`\W` (No caracteres alfanuméricos)](#w-no-caracteres-alfanuméricos)
  - [`\s` (Espacios en blanco)](#s-espacios-en-blanco)
  - [`\S` (No espacios en blanco)](#s-no-espacios-en-blanco)
  - [`.` (Cualquier carácter)](#-cualquier-carácter)

## Caracteres

Las expresiones regulares son herramientas poderosas en JavaScript para la búsqueda y manipulación de patrones en cadenas de texto. Uno de los conceptos más útiles en las expresiones regulares es el uso de caracteres y clases de caracteres, que permiten especificar conjuntos de caracteres que deben coincidir en determinadas posiciones dentro de una cadena.

Este capítulo cubrirá las clases de caracteres, cómo se utiliza la negación dentro de estas clases, y las clases predefinidas que se pueden usar para facilitar la creación de expresiones regulares.

## Clases de Caracteres (`[...]`)

Una clase de caracteres en una expresión regular se utiliza para definir un conjunto de caracteres entre los que debe coincidir una posición específica de la cadena. Se representa entre corchetes `[...]`, y cada carácter dentro de los corchetes es una posible coincidencia.

```js
let regex = /[aeiou]/;
console.log(regex.test("hola")); // true (porque "o" es una vocal)
console.log(regex.test("xyz")); // false (porque no contiene ninguna vocal)
```

En este caso, la expresión regular `[aeiou]` coincide con cualquier vocal minúscula (a, e, i, o, u). Si la cadena contiene alguna de estas letras, se considera una coincidencia.

### Ejemplo con rangos de caracteres

Además de enumerar caracteres individuales, se pueden utilizar rangos dentro de las clases de caracteres usando guiones (`-`). Por ejemplo, `[a-z]` representa todas las letras minúsculas, mientras que `[0-9]` representa todos los dígitos

```js
let regex = /[a-z]/; // Coincide con cualquier letra minúscula
console.log(regex.test("Hola")); // true (porque "o" está en el rango a-z)

let regexNumeros = /[0-9]/; // Coincide con cualquier dígito
console.log(regexNumeros.test("123abc")); // true (contiene dígitos)
```

## Negación de Clases de Caracteres (`[^...]`)

Para negar una clase de caracteres, es decir, para especificar los caracteres con los que no debería coincidir una posición, se utiliza el símbolo ^ justo después del corchete de apertura `[`. Esto indica que la expresión debe coincidir con cualquier carácter excepto los incluidos en la clase.

```js
let regex = /[^aeiou]/; // Coincide con cualquier carácter que no sea una vocal
console.log(regex.test("hola")); // true (porque "h" no es vocal)
console.log(regex.test("aeiou")); // false (todos son vocales)
```

En este caso, `[^aeiou]` coincidirá con cualquier carácter que no sea una vocal minúscula.

### Ejemplo con rango de caracteres negados

```js
let regex = /[^0-9]/; // Coincide con cualquier carácter que no sea un dígito
console.log(regex.test("123")); // false (solo dígitos)
console.log(regex.test("abc123")); // true (porque contiene letras)
```

## Clases Predefinidas

Para simplificar la creación de expresiones regulares, JavaScript proporciona una serie de clases predefinidas que representan grupos comunes de caracteres. Estas clases son abreviaciones que permiten describir tipos de caracteres frecuentes, como dígitos o espacios en blanco, de manera más concisa.

### `\d` (Dígitos)

`\d` es una abreviatura para cualquier dígito (equivalente a `[0-9]`).

```js
let regex = /\d/; // Coincide con cualquier dígito
console.log(regex.test("123")); // true
console.log(regex.test("abc")); // false
```

### `\D` (No dígitos)

`\D` coincide con cualquier carácter que no sea un dígito (equivalente a `[^0-9]`).

```js
let regex = /\D/; // Coincide con cualquier carácter que no sea un dígito
console.log(regex.test("abc")); // true
console.log(regex.test("123")); // false
```

### `\w` (Caracteres alfanuméricos)

`\w` coincide con cualquier carácter alfanumérico (letras, dígitos y el guion bajo _). Es equivalente a ``[A-Za-z0-9_]``.

```js
let regex = /\w/; // Coincide con letras, números y guion bajo
console.log(regex.test("abc123_")); // true
console.log(regex.test("!@#")); // false
```

### `\W` (No caracteres alfanuméricos)

`\W` coincide con cualquier carácter que no sea alfanumérico (equivalente a `[^A-Za-z0-9_]`).

```js
let regex = /\W/; // Coincide con caracteres que no son letras, números ni guion bajo
console.log(regex.test("abc123")); // false
console.log(regex.test("!@#")); // true
```

### `\s` (Espacios en blanco)

`\s` coincide con cualquier espacio en blanco, incluidos el espacio, tabulación, salto de línea y otros caracteres de espacio (equivalente a `[ \t\r\n\f\v]`).

```js
let regex = /\s/; // Coincide con cualquier espacio en blanco
console.log(regex.test("hola mundo")); // true (porque hay un espacio entre "hola" y "mundo")
console.log(regex.test("holamundo")); // false
```

### `\S` (No espacios en blanco)

`\S` coincide con cualquier carácter que no sea un espacio en blanco (equivalente a `[^ \t\r\n\f\v]`).

```js
let regex = /\S/; // Coincide con cualquier carácter que no sea un espacio en blanco
console.log(regex.test("hola mundo")); // true (porque "hola" y "mundo" no son espacios)
console.log(regex.test(" ")); // false (es un espacio en blanco)
```

### `.` (Cualquier carácter)

El punto (`.`) coincide con cualquier carácter excepto un salto de línea (`\n`). Es una clase comodín muy utilizada en expresiones regulares.

```js
let regex = /./; // Coincide con cualquier carácter, excepto una nueva línea
console.log(regex.test("hola")); // true
console.log(regex.test(" ")); // true
console.log(regex.test("\n")); // false (porque es un salto de línea)
```

Si se desea que el punto también coincida con saltos de línea, se debe habilitar la bandera `s` (modo de punto dotAll) en la expresión regular.

```js
let regex = /./s; // Coincide con cualquier carácter, incluyendo nueva línea
console.log(regex.test("\n")); // true
```
