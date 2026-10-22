<h1 align='center'>Conceptos Clave</h1>

<h2>📑 Contenido</h2>

- [Conceptos Clave](#conceptos-clave)
- [Hoisting (Elevación)](#hoisting-elevación)
- [Scope (Ámbito)](#scope-ámbito)
  - [Ámbito Global](#ámbito-global)
  - [Ámbito Local (Funcional)](#ámbito-local-funcional)
  - [Ámbito de Bloque:](#ámbito-de-bloque)
- [Call Stack (Pila de Llamadas)](#call-stack-pila-de-llamadas)
- [Closures (Clausuras)](#closures-clausuras)
- [Contexto Léxico](#contexto-léxico)

## Conceptos Clave

JavaScript es un lenguaje que, aunque relativamente fácil de aprender en sus principios básicos, encierra conceptos fundamentales que todo programador debe dominar para trabajar eficientemente y comprender cómo el lenguaje maneja la ejecución de código. En este capítulo, exploraremos y reforzaremos algunos de los conceptos más importantes, como el hoisting, el scope, el call stack, las closures, los callbacks, y otros términos clave que forman parte del corazón de JavaScript.

## Hoisting (Elevación)

El hoisting (o "elevación") es un comportamiento en JavaScript donde las declaraciones de variables y funciones se "eleva" al principio de su contexto de ejecución (ya sea un bloque, función o script) antes de que el código sea ejecutado.

Cuando JavaScript se ejecuta, primero revisa todas las declaraciones de variables y funciones en el código y las eleva (o "hoistea") al inicio del contexto. Esto significa que puedes usar variables y funciones antes de haberlas declarado en el código.

```js
console.log(x); // undefined
var x = 5;
```

En este ejemplo, la declaración `var x` es "elevada" al principio del contexto, pero no la asignación, por lo que el valor inicial de `x` es `undefined`.

En el caso de las funciones declaradas con `function`, la declaración completa (tanto el nombre como la implementación) se eleva, lo que significa que puedes invocar la función antes de su declaración.

```js
saludar(); // Funciona aunque la función está declarada después

function saludar() {
  console.log("Hola");
}
```

Es importante notar que `let` y `const` también son elevadas, pero no están inicializadas. Acceder a estas variables antes de su declaración resultará en un error.

## Scope (Ámbito)

El scope o ámbito define dónde son accesibles las variables en tu código. En JavaScript, existen varios tipos de ámbito:

### Ámbito Global

Una variable definida en el ámbito global es accesible en cualquier parte del código. En un archivo de script, las variables declaradas fuera de cualquier función o bloque pertenecen al ámbito global.

```js
var globalVar = "Soy global";

function mostrarGlobal() {
  console.log(globalVar); // Puede acceder a `globalVar`
}
```

### Ámbito Local (Funcional)

Una variable definida dentro de una función tiene un ámbito local y no es accesible fuera de la función.

```js
function localScope() {
  var localVar = "Soy local";
  console.log(localVar); // Visible dentro de la función
}

console.log(localVar); // Error, `localVar` no está definida aquí
```

### Ámbito de Bloque:

Con `let` y `const`, JavaScript introduce el ámbito de bloque, que limita la visibilidad de una variable a un bloque de código entre llaves (`{}`), como en estructuras `if`, `for`, o funciones.

```js
if (true) {
  let bloque = "Dentro del bloque";
  console.log(bloque); // Funciona
}

console.log(bloque); // Error, `bloque` no está accesible fuera
```

## Call Stack (Pila de Llamadas)

El call stack es una estructura de datos que rastrea el orden de ejecución de las funciones en JavaScript. Cada vez que se invoca una función, se coloca en la cima de la pila. Cuando una función completa su ejecución, se elimina de la pila. Si una función llama a otra función, esta última se coloca en la cima del call stack.

```js
function primera() {
  console.log("Primera función");
  segunda();
}

function segunda() {
  console.log("Segunda función");
}

primera();
```

En este ejemplo:

1. primera() se coloca en la pila.
1. Durante la ejecución de primera(), se llama a segunda(), que se coloca en la cima del call stack.
1. Cuando segunda() termina, se elimina de la pila.
1. Luego, primera() termina y también se elimina.

## Closures (Clausuras)

Un closure es una función que recuerda su entorno léxico, es decir, recuerda el ámbito en el que fue creada, incluso después de que ese ámbito haya finalizado. Las closures permiten acceder a las variables de una función externa desde una función interna, incluso después de que la función externa haya terminado de ejecutarse.

```js
function crearContador() {
  let contador = 0;
  return function () {
    contador++;
    return contador;
  };
}

let contar = crearContador();
console.log(contar()); // 1
console.log(contar()); // 2
```

En este ejemplo, la función interna devuelta sigue teniendo acceso a la variable contador de la función externa, aunque esta última haya terminado de ejecutarse.

## Contexto Léxico

El contexto léxico en JavaScript se refiere al ámbito en el que se definió una función o variable, no necesariamente donde se ejecuta. Las funciones en JavaScript siempre recuerdan el contexto léxico en el que fueron creadas.

```js
function fuera() {
  let x = 10;

  function dentro() {
    console.log(x); // `x` se recuerda del contexto léxico donde fue creada
  }

  dentro();
}

fuera(); // Imprime 10
```

Estos son algunos de los conceptos clave que forman la base del funcionamiento de JavaScript. Dominar estos temas te ayudará a escribir código más eficiente, comprensible y robusto.
