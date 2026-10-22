<h1 align='center'>Conversión de Tipo</h1>

<h2>📑 Contenido</h2>

- [Conversión de Tipo](#conversión-de-tipo)
- [Conversión Implícita (Coerción de Tipo)](#conversión-implícita-coerción-de-tipo)
  - [Coerción a String](#coerción-a-string)
  - [Coerción a Number](#coerción-a-number)
  - [Coerción a Boolean](#coerción-a-boolean)
- [Conversión Explícita](#conversión-explícita)
  - [Conversión a String](#conversión-a-string)
  - [Conversión a Number](#conversión-a-number)
  - [Conversión a Boolean](#conversión-a-boolean)
- [Curiosidades y Peligros de la Coerción](#curiosidades-y-peligros-de-la-coerción)
  - [El problema del operador +](#el-problema-del-operador-)
  - [parseInt() y números con prefijos](#parseint-y-números-con-prefijos)
  - [Conversión en Comparaciones Lógicas](#conversión-en-comparaciones-lógicas)

## Conversión de Tipo

En JavaScript, los valores pueden cambiar de un tipo de dato a otro de manera implícita o explícita, lo que se conoce como conversión de tipo. Este proceso ocurre de manera automática (conversión implícita o coerción de tipo) o manualmente mediante la intervención del programador (conversión explícita). Dado que JavaScript es un lenguaje de tipado dinámico, esta característica es crucial para entender cómo se manipulan los datos en diferentes contextos.

## Conversión Implícita (Coerción de Tipo)

La conversión implícita es cuando JavaScript convierte automáticamente un tipo de dato en otro según lo requiera el contexto. Esto puede ocurrir en operaciones aritméticas, comparaciones, concatenaciones y otras interacciones entre valores de distintos tipos.

### Coerción a String

Cuando un valor que no es una cadena se concatena con una cadena (+ en JavaScript), el valor se convierte en un String.

```js
let numero = 5;
let texto = "El número es: " + numero;
console.log(texto); // "El número es: 5"
```

En este caso, el número 5 se convierte en una cadena "5" para que pueda ser concatenado con el texto.

### Coerción a Number

En operaciones aritméticas (excepto + cuando está involucrada una cadena), JavaScript intenta convertir las cadenas a números.

```js
let resultado = "10" - 5;
console.log(resultado); // 5
```

Aquí, la cadena "10" se convierte implícitamente en el número 10 para que la resta pueda ser realizada.

### Coerción a Boolean

JavaScript convierte algunos valores a boolean de manera implícita en contextos lógicos como en las condiciones de los if o los bucles que veremos en control de flujo. Los siguientes valores son `falsy` (se consideran `false` en un contexto booleano):

Falsy:

- false
- 0
- "" (cadena vacía)
- null
- undefined
- NaN

Cualquier otro valor se considera `truthy` y se evalúa como `true`.

## Conversión Explícita

La conversión explícita ocurre cuando el programador usa métodos específicos para cambiar un valor de un tipo a otro. Es una forma controlada de asegurarse de que los valores sean del tipo correcto antes de realizar operaciones

### Conversión a String

Se puede convertir cualquier valor a una cadena utilizando el método `String()` o simplemente concatenando con una cadena vacía ("").

```js
let numero = 123;
let texto = String(numero); // "123"
console.log(typeof texto); // "string"

let otraConversion = numero + ""; // "123"
console.log(otraConversion); // "123"
```

### Conversión a Number

Para convertir explícitamente un valor a número, se puede utilizar la función `Number()`, el operador unario +, o las funciones `parseInt()` y `parseFloat()` (para enteros y números decimales respectivamente).

```js
let texto = "456";
let numero = Number(texto); // 456
console.log(numero); // 456

let otraConversion = +"123"; // 123 (con el operador unario +)
console.log(otraConversion); // 123

let decimal = parseFloat("3.14"); // 3.14
console.log(decimal); // 3.14

let entero = parseInt("42"); // 42
console.log(entero); // 42
```

### Conversión a Boolean

Cualquier valor se puede convertir explícitamente a un valor booleano utilizando la función `Boolean()`. Esto es útil cuando se desea comprobar si un valor es `truthy` o `falsy` de manera explícita.

## Curiosidades y Peligros de la Coerción

### El problema del operador +

El operador + puede ser confuso en JavaScript porque actúa como operador aritmético y como operador de concatenación. Dependiendo de si uno de los operandos es una cadena, el comportamiento puede cambiar.

```js
let resultado = "5" + 10;
console.log(resultado); // "510" (concatenación)

let otroResultado = 5 + "10";
console.log(otroResultado); // "510" (concatenación)
```

Aquí, aunque en ambos casos hay un número involucrado, debido a que uno de los operandos es una cadena, JavaScript convierte el número en cadena y luego concatena.

### parseInt() y números con prefijos

Cuando `parseInt()` encuentra un número que comienza con `0x`, lo interpreta como un número en base 16 (hexadecimal). Esto puede generar resultados inesperados si no se tiene en cuenta.

```js
let hex = "0x10";
let numero = parseInt(hex);
console.log(numero); // 16, porque "0x10" es 16 en base 16
```

Para evitar confusiones, es posible pasar el segundo argumento a parseInt(), que define la base del número.

```js
let decimal = parseInt("10", 10); // Interpreta el número en base 10
console.log(decimal); // 10
```

### Conversión en Comparaciones Lógicas

Cuando se usa una comparación, como` <, >, <= o >=`, JavaScript convierte los valores en números (si es posible) para hacer la evaluación.

```js
console.log("5" < 10); // true, la cadena "5" se convierte en el número 5
console.log("apple" < "banana"); // true, comparación lexicográfica
```
