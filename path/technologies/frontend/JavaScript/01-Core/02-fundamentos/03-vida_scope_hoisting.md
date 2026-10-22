<h1 align='center'>Vida, Scope y Hoisting de la variables</h1>

<h2>📑 Contenido</h2>

- [Vida de las Variables](#vida-de-las-variables)
- [Scope (Ámbito) de las Variables](#scope-ámbito-de-las-variables)
  - [Scope global](#scope-global)
  - [Scope local (de función)](#scope-local-de-función)
  - [Scope de bloque (desde ES6)](#scope-de-bloque-desde-es6)
- [Hoisting](#hoisting)
  - [Hoisting en variables](#hoisting-en-variables)
  - [Hoisting con let y const](#hoisting-con-let-y-const)
  - [Hoisting en funciones](#hoisting-en-funciones)

## Vida de las Variables

La "vida" de una variable en programación se refiere a cuánto tiempo existe en la memoria del programa antes de que sea destruida o ya no esté disponible. En JavaScript, el ciclo de vida de una variable depende de dónde y cómo ha sido declarada.

**Variables globales:** Se declaran fuera de cualquier función o bloque. Su ciclo de vida empieza cuando se carga la página (o el script), y terminan cuando la página o el script finaliza su ejecución.

**Variables locales:** Se declaran dentro de funciones o bloques de código. Su ciclo de vida comienza cuando se ejecuta la función o el bloque y termina cuando dicha ejecución finaliza.

```js
var globalVar = "Soy una variable global";

function ejemplo() {
  var localVar = "Soy una variable local";
  console.log(globalVar); // Puede acceder a la variable global.
  console.log(localVar); // Puede acceder a la variable local.
}

ejemplo(); // Aquí la variable localVar aún existe.

console.log(globalVar); // Funciona.
console.log(localVar); // Error, ya que localVar solo vive dentro de la función.
```

## Scope (Ámbito) de las Variables

El scope o ámbito de una variable define en qué partes del código la variable es accesible. JavaScript tiene dos tipos principales de scope:

### Scope global

Las variables declaradas en el scope global son accesibles desde cualquier parte del código. En general, este tipo de variables deben usarse con cuidado, ya que una variable global puede ser accidentalmente sobrescrita por otras partes del programa.

```js
var globalVar = "Estoy en el scope global";

function mostrarGlobal() {
  console.log(globalVar); // Puede acceder a globalVar
}

mostrarGlobal(); // "Estoy en el scope global"
```

### Scope local (de función)

Las variables declaradas dentro de una función solo pueden ser accedidas dentro de esa función, no fuera de ella.

```js
function ejemploLocal() {
  var localVar = "Estoy en el scope local";
  console.log(localVar); // Accesible dentro de la función
}

ejemploLocal(); // "Estoy en el scope local"
console.log(localVar); // Error, no se puede acceder fuera de la función.
```

### Scope de bloque (desde ES6)

A partir de ES6, las variables declaradas con let o const tienen un scope de bloque. Un bloque es cualquier código entre llaves {}, como un if, for o while. Esto significa que las variables creadas con let o const solo son accesibles dentro de ese bloque específico.

```js
if (true) {
  let bloqueVar = "Estoy en el scope de bloque";
  console.log(bloqueVar); // Funciona dentro del bloque
}

console.log(bloqueVar); // Error, no se puede acceder fuera del bloque
```

## Hoisting

El hoisting (elevación) es un comportamiento en JavaScript donde las declaraciones de variables y funciones son "elevadas" al principio de su scope antes de que el código se ejecute. Esto significa que puedes usar variables y funciones antes de haberlas declarado, aunque esto puede causar confusión y errores si no se entiende correctamente.

### Hoisting en variables

Cuando declaras una variable con `var`, su declaración se "eleva" al principio del scope, pero su inicialización no. Esto significa que puedes referenciar la variable antes de su declaración, aunque su valor será `undefined` hasta que se le asigne un valor explícitamente.

```js
console.log(x); // undefined
var x = 5;
console.log(x); // 5
```

Lo que realmente ocurre es que JavaScript lo interpreta así:

```js
var x;
console.log(x); // undefined
x = 5;
console.log(x); // 5
```

### Hoisting con let y const

Las variables declaradas con `let` o `const` también son elevadas, pero no pueden ser accedidas antes de la declaración. Si intentas hacerlo, obtendrás un error de referencia. Esto se conoce como zona temporalmente muerta.

```js
console.log(y); // Error: Cannot access 'y' before initialization
let y = 10;
```

### Hoisting en funciones

> [!NOTE]
> Una función es un bloque de código que realiza una tarea específica y puede ser reutilizado en diferentes partes de un programa. Una función toma entradas (parámetros), realiza una serie de acciones y, opcionalmente, devuelve un resultado.

Las funciones declaradas usando function son completamente elevadas al principio del scope, lo que significa que puedes llamarlas incluso antes de su declaración en el código.

```js
saludo(); // Funciona perfectamente

function saludo() {
  console.log("¡Hola!");
}
```

Pero si las funciones se asignan a una variable con `var`, `let` o `const`, entonces el hoisting se comporta como el de las variables, y la función no se puede llamar antes de su declaración:

```js
// Ejemplo con var:
console.log(saludar); // undefined, ya que la variable se eleva pero no la función
var saludar = function () {
  console.log("¡Hola!");
};

// Ejemplo con let o const:
console.log(saludar); // Error: Cannot access 'saludar' before initialization
let saludar = function () {
  console.log("¡Hola!");
};
```
