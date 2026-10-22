<h1 align='center'>Funciones Integradas</h1>

<h2>📑 Contenido</h2>

- [Funciones Integradas](#funciones-integradas)
- [¿Qué son las funciones integradas?](#qué-son-las-funciones-integradas)
- [Funciones Globales Comunes](#funciones-globales-comunes)
  - [parseInt()](#parseint)
  - [parseFloat()](#parsefloat)
  - [isNaN()](#isnan)
  - [Number()](#number)
  - [eval()](#eval)

## Funciones Integradas

JavaScript incluye una amplia gama de funciones integradas o built-in functions, que son funciones predefinidas en el lenguaje. Estas funciones están disponibles globalmente o como parte de objetos nativos y proporcionan una base sólida para realizar tareas comunes como manipulación de datos, ejecución de cálculos matemáticos, manipulación de cadenas de texto, y manejo de fechas, entre otras.

Este capítulo cubre algunas de las funciones integradas más importantes de JavaScript, cómo se utilizan, y cómo pueden ayudarte a escribir código más eficiente y confiable.

## ¿Qué son las funciones integradas?

Las funciones integradas en JavaScript son funciones predefinidas que vienen con el lenguaje y están disponibles sin necesidad de que el desarrollador las implemente. Estas funciones ayudan a realizar tareas comunes de forma sencilla y eficiente, evitando la necesidad de escribir código adicional para operaciones que se utilizan frecuentemente.

Existen funciones globales como parseInt(), parseFloat() y isNaN(), y también funciones que pertenecen a objetos integrados como Math, String, Array, Date, entre otros.

> [!NOTE]
> Las funciones que pertenecen a objetos incorporados se verán en su sección.

## Funciones Globales Comunes

JavaScript incluye varias funciones que son accesibles de forma global, sin necesidad de ser parte de un objeto en particular. Algunas de las más comunes incluyen:

### parseInt()

Convierte una cadena de texto en un número entero.

```js
let numero = parseInt("123"); // Devuelve 123
let hexadecimal = parseInt("A", 16); // Devuelve 10 (base hexadecimal)
```

### parseFloat()

Convierte una cadena de texto en un número con punto flotante.

```js
let decimal = parseFloat("3.14"); // Devuelve 3.14
```

### isNaN()

Determina si un valor es NaN ("Not a Number"), un valor especial en JavaScript que indica que algo no es numéricamente válido.

```js
let resultado = isNaN("abc"); // Devuelve true
```

### Number()

Convierte un valor a tipo número.

```js
let numero = Number("456"); // Devuelve 456
```

### eval()

Evalúa una cadena de texto como código JavaScript. Aunque esta función puede parecer poderosa, debe usarse con precaución por razones de seguridad, ya que puede ejecutar código malicioso si no se controla adecuadamente.

```js
eval("2 + 2"); // Devuelve 4
```

> [!WARNING]
> La función`eval()` es considerada "peligrosa" y no es recomendada en la mayoría de los casos por posibles riesgos de seguridad.
