<h1 align='center'>Tipos de datos (Primitivos)</h1>

<h2>📑 Contenido</h2>

- [Tipos de Datos](#tipos-de-datos)
- [Tipos de Datos Primitivos](#tipos-de-datos-primitivos)
  - [Number (Números)](#number-números)
  - [String (Cadenas de Texto)](#string-cadenas-de-texto)
  - [Boolean (Booleanos)](#boolean-booleanos)
  - [Undefined (Indefinido)](#undefined-indefinido)
  - [Null (Nulo)](#null-nulo)
  - [Symbol (Símbolos, introducido en ES6)](#symbol-símbolos-introducido-en-es6)
- [Curiosidades](#curiosidades)
  - [undefined es un valor y tipo](#undefined-es-un-valor-y-tipo)
  - [null es un objeto (bug histórico)](#null-es-un-objeto-bug-histórico)
  - [Comparación null y undefined](#comparación-null-y-undefined)
  - [NaN: Not-a-Number, pero sigue siendo un número](#nan-not-a-number-pero-sigue-siendo-un-número)
  - [NaN no es igual a sí mismo](#nan-no-es-igual-a-sí-mismo)
  - [Number.isNaN() vs isNaN()](#numberisnan-vs-isnan)
- [Valores Truthy y Falsy](#valores-truthy-y-falsy)
  - [Valores Falsy](#valores-falsy)
  - [Valores Truthy](#valores-truthy)

## Tipos de Datos

Los tipos de datos son fundamentales para entender cómo se almacenan y manipulan los valores dentro del programa. Este lenguaje es de tipado dinámico, lo que significa que no es necesario declarar el tipo de una variable explícitamente, ya que JavaScript puede inferirlo en tiempo de ejecución. En JavaScript están los tipos de datos primitivos(Number, String, Boolean, Undefined, Nul, Symbol) y los compuestos(Objetos, Funciones, Arrays), estos últimos tendrán cada uno su propia sección.

## Tipos de Datos Primitivos

Los tipos primitivos en JavaScript son aquellos que no son objetos y representan un solo valor. Existen seis tipos de datos primitivos:

### Number (Números)

Este tipo representa tanto números enteros como números decimales (números de punto flotante).

```js
let edad = 30; // Número entero
let temperatura = 36.5; // Número decimal
```

Además, JavaScript soporta valores especiales dentro de los números:

**Infinity:** Resultado de una división por cero o de un cálculo que excede los límites del lenguaje.

**NaN (Not-a-Number):** Indica que el resultado de una operación no es un número válido.

```js
let infinito = 1 / 0; // Infinity
let noEsNumero = "Hola" * 3; // NaN
```

### String (Cadenas de Texto)

Las cadenas son secuencias de caracteres y pueden definirse entre comillas simples ('...'), dobles ("..."), o plantillas literales con backticks (\`...\`), que permiten interpolación de variables.

```js
let saludo = "Hola, mundo"; // Comillas dobles
let mensaje = "JavaScript es genial"; // Comillas simples
let nombre = `Juan`; // Template literal

// Interpolación de variables en plantillas literales
let edad = 25;
let frase = `Tengo ${edad} años`; // "Tengo 25 años"
```

### Boolean (Booleanos)

Un valor booleano solo puede ser true o false. Se utilizan principalmente en expresiones condicionales.

```js
let esMayorDeEdad = true;
let estaLloviendo = false;
```

### Undefined (Indefinido)

Cuando una variable se declara pero no se le asigna ningún valor, su tipo es `undefined`. También puede ser el valor de retorno de una función que no tiene una instrucción `return`.

```js
let sinValor;
console.log(sinValor); // undefined
```

### Null (Nulo)

El tipo `null` representa la ausencia intencional de cualquier valor u objeto. A diferencia de `undefined`, `null` es asignado explícitamente por el programador para indicar "sin valor".

```js
let coche = null; // Se asigna null para indicar que no hay coche
```

### Symbol (Símbolos, introducido en ES6)

Un `Symbol` es un valor único e inmutable que se puede utilizar como identificador para las propiedades de un objeto. Cada símbolo es único, incluso si se crean con el mismo valor.

```js
let id1 = Symbol("id");
let id2 = Symbol("id");
console.log(id1 === id2); // false, cada símbolo es único
```

## Curiosidades

JavaScript es un lenguaje lleno de peculiaridades, y tres de los conceptos que pueden confundir a los desarrolladores, especialmente a quienes están empezando, son null, NaN y undefined. Aunque estos tres valores parecen similares en algunos contextos (por representar "ausencia" o "no ser un valor válido"), tienen comportamientos, usos y curiosidades que los distinguen.

### undefined es un valor y tipo

El operador `typeof` (indica de que tipo es) aplicado a una variable undefined siempre retornará "`undefined`", lo que indica que `undefined` es tanto el valor como el tipo de la variable.

```js
let b;
console.log(typeof b); // "undefined"
```

### null es un objeto (bug histórico)

Una curiosidad famosa de JavaScript es que, si utilizamos `typeof` para comprobar el tipo de `null`, obtendremos "`object`". Esto es un error histórico en el lenguaje, pero se ha mantenido por razones de compatibilidad con versiones anteriores.

```js
console.log(typeof null); // "object"
```

Este comportamiento es consecuencia de cómo JavaScript gestiona los valores en memoria. Inicialmente, los valores que ocupaban un espacio en memoria (objetos) se distinguían de los primitivos, y null fue categorizado erróneamente como un objeto en su origen. Aunque no lo es, este comportamiento persiste.

### Comparación null y undefined

Aunque null y undefined parecen intercambiables en algunos casos, hay diferencias clave:

- `null` es algo que el desarrollador asigna explícitamente para indicar "sin valor".
- `undefined` se produce cuando algo no ha sido asignado.

Al usar el operador de igualdad débil (==), JavaScript considera null y undefined como equivalentes. Pero con el operador de igualdad estricta (===), que compara tanto el tipo como el valor, no son iguales.

```js
console.log(null == undefined); // true
console.log(null === undefined); // false
```

### NaN: Not-a-Number, pero sigue siendo un número

`NaN` es un valor especial en JavaScript que representa un valor que no es un número válido. Aparece principalmente cuando una operación aritmética no tiene sentido, como intentar multiplicar una cadena que no contiene un número con un valor numérico, o cuando una operación matemática no puede ser calculada.

Aunque parece contradictorio, el tipo de NaN es number. Esto es porque NaN es un valor numérico especial que indica la falta de un valor numérico válido, pero sigue perteneciendo al tipo numérico.

```js
console.log(typeof NaN); // "number"
```

### NaN no es igual a sí mismo

Una de las curiosidades más inusuales de `NaN` es que es el único valor en JavaScript que no es igual a sí mismo. Esto significa que si comparas `NaN` con `NaN`, el resultado será false.

```js
console.log(NaN === NaN); // false
```

Esta característica se debe a que NaN representa un valor que no es numéricamente válido, y por lo tanto no puede considerarse igual a otro valor, ni siquiera a sí mismo. Para verificar si un valor es NaN, se debe usar la función isNaN():

```js
console.log(isNaN(NaN)); // true
```

### Number.isNaN() vs isNaN()

Hay una diferencia entre `isNaN()` y` Number.isNaN()`. La función `isNaN()` realiza una conversión implícita antes de verificar si el valor es NaN, lo que puede llevar a resultados inesperados:

```js
console.log(isNaN("hola")); // true, convierte "hola" a NaN
```

Por otro lado, `Number.isNaN()` no realiza ninguna conversión, por lo que solo retornará true si el valor pasado es exactamente `NaN`:

```js
console.log(Number.isNaN("hola")); // false, porque "hola" no es un NaN
console.log(Number.isNaN(NaN)); // true
```

## Valores Truthy y Falsy

Cualquier valor puede evaluarse en un contexto booleano (como en una condición `if`). Según su naturaleza, se comporta como true (`truthy`) o false (`falsy`).

### Valores Falsy

Son los que se consideran `false` en un contexto booleano:

- `false`
- `0, -0`
- `0n (BigInt cero)`
- `"" (cadena vacía)`
- `null`
- `undefined`
- `NaN`
- `document.all (caso especial para compatibilidad)`

```javascript
if (0) {
  console.log("Esto no se ejecuta");
}
```

### Valores Truthy

Todo lo que no sea `falsy` se considera `truthy`:

- `Números distintos de cero (42, -1)`
- `Cadenas no vacías ("hola", "false")`
- `Objetos ({}, [])`
- `Funciones`
- `Fechas (new Date())`
- `Symbol()`

```javascript
if ("JavaScript") {
  console.log("¡Esto sí se ejecuta!");
}
```
