<h1 align='center'>Funciones Expresadas</h1>

<h2>📑 Contenido</h2>

- [Funciones Expresadas](#funciones-expresadas)
- [¿Qué es una función expresada?](#qué-es-una-función-expresada)
- [Funciones expresadas vs Declaraciones de funciones](#funciones-expresadas-vs-declaraciones-de-funciones)
  - [Declarada](#declarada)
  - [Expresada](#expresada)
- [Funciones expresadas como valores](#funciones-expresadas-como-valores)
  - [En Variables](#en-variables)
  - [En Arrays](#en-arrays)
  - [En Objetos](#en-objetos)

## Funciones Expresadas

Las funciones expresadas (o function expressions en inglés) son una de las dos formas principales de definir funciones en JavaScript. A diferencia de las declaraciones de funciones, las funciones expresadas se definen dentro de una expresión o asignación. Esto significa que se pueden asignar a variables, pasar como argumentos a otras funciones y, en general, tratarlas como valores.

En este capítulo, profundizaremos en cómo funcionan las funciones expresadas, cómo se comportan en comparación con las declaraciones de funciones, y cuándo es útil emplearlas en tu código.

## ¿Qué es una función expresada?

Una función expresada es una función que se define como parte de una expresión o una asignación. Generalmente, se asigna a una variable para que se pueda invocar posteriormente, aunque también puede ser anónima. La diferencia clave con las declaraciones de funciones es que las funciones expresadas no se elevan (no tienen hoisting), lo que significa que solo están disponibles después de ser definidas.

> [!NOTE]
> El **hoisting** es un comportamiento donde las declaraciones de variables y funciones son "elevadas" o "movidas" a la parte superior de su contexto de ejecución antes de que el código sea ejecutado. Esto significa que puedes utilizar funciones y variables antes de haberlas declarado en el código.

```js
// Sintaxis de una función expresada:

const saludar = function () {
  console.log("¡Hola!");
};

saludar(); // Invoca la función
```

En este ejemplo, la función anónima se asigna a la variable saludar, y se puede invocar utilizando el nombre de la variable, tal como se haría con cualquier función.

## Funciones expresadas vs Declaraciones de funciones

La diferencia principal entre una función expresada y una declaración de función es cómo y cuándo están disponibles en el código.

### Declarada

En una declaración de función, la función se eleva (hoisting), lo que significa que puedes llamarla antes de que aparezca en el código. Esto se debe a que JavaScript mueve las declaraciones de funciones al principio del contexto de ejecución.

```js
// Declaración de función - Esto funcionará debido al hoisting
saludar();

function saludar() {
  console.log("¡Hola!");
}
```

### Expresada

En el caso de una función expresada, no se puede invocar antes de que se defina, ya que no se eleva. Si intentas llamar a la función antes de su definición, obtendrás un error.

```js
// Función expresada - Esto fallará, ya que no está definida aún
// saludar();  // Error: saludar is not a function

const saludar = function () {
  console.log("¡Hola!");
};
```

## Funciones expresadas como valores

Las funciones en JavaScript son ciudadanos de primera clase (first-class citizens), lo que significa que pueden ser tratadas como cualquier otro valor, como strings o números. Esto implica que se pueden asignar a variables, almacenar en estructuras de datos como arrays u objetos, y pasar como argumentos a otras funciones.

### En Variables

```js
const multiplicar = function (a, b) {
  return a * b;
};

console.log(multiplicar(2, 4)); // Resultado: 8
```

### En Arrays

```js
const operaciones = [
  function (a, b) {
    return a + b;
  },
  function (a, b) {
    return a - b;
  },
  function (a, b) {
    return a * b;
  },
];

console.log(operaciones[0](4, 2)); // Resultado: 6 (suma)
console.log(operaciones[1](4, 2)); // Resultado: 2 (resta)
```

### En Objetos

```js
const calculadora = {
  sumar: function (a, b) {
    return a + b;
  },
  restar: function (a, b) {
    return a - b;
  },
};

console.log(calculadora.sumar(3, 5)); // Resultado: 8
console.log(calculadora.restar(10, 4)); // Resultado: 6
```

En estos ejemplos, las funciones se comportan como cualquier otro valor y pueden ser almacenadas y manipuladas dentro de arrays y objetos que veremos en las siguientes secciones.
