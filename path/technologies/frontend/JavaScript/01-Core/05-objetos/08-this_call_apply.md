<h1 align="center">This, Call y Apply</h1>

<h2>📑 Contenido</h2>

- [this](#this)
  - [Contextos comunes de this](#contextos-comunes-de-this)
- [call() y apply()](#call-y-apply)
  - [call(thisArg, arg1, arg2, ...)](#callthisarg-arg1-arg2-)
  - [apply(thisArg, \[argsArray\])](#applythisarg-argsarray)
- [Resumen](#resumen)

## this

La palabra clave `this` hace referencia al objeto en el que se está ejecutando el código. Su valor no es fijo; depende del contexto en el que se invoca la función. Esta flexibilidad es una de las partes más confusas de JavaScript para los principiantes.

### Contextos comunes de this

En el ámbito global:

Fuera de cualquier función, `this` se refiere al objeto global (`window` en el navegador o ''global'' en Node.js).

```javascript
console.log(this === window); // true
```

Como método de un objeto:

Cuando una función se llama como un método de un objeto, `this` se refiere a ese objeto.

```javascript
const persona = {
  nombre: "Ana",
  saludar() {
    console.log(`Hola, soy ${this.nombre}`); // this se refiere a `persona`
  },
};
persona.saludar(); // "Hola, soy Ana"
```

En una función simple:

En el modo no estricto, `this` se refiere al objeto `global`. En el modo estricto (`"use strict";`), `this` es `undefined`.

```javascript
function saludar() {
  console.log(this); // window (en modo no estricto)
}
saludar();
```

En una función constructora (`new`): Cuando una función se invoca con la palabra clave `new`, this se refiere a la nueva instancia del objeto que se está creando.

```javascript
function Coche(marca) {
  this.marca = marca; // this se refiere a la nueva instancia
}
const miCoche = new Coche("Toyota");
console.log(miCoche.marca); // "Toyota"
```

## call() y apply()

Estos dos métodos son funciones que se encuentran en el prototipo de `Function` y te permiten invocar una función y establecer el valor de `this` de forma explícita. La principal diferencia está en cómo pasan los argumentos a la función.

### call(thisArg, arg1, arg2, ...)

Invoca una función con un valor de `this` específico y argumentos que se pasan individualmente.

Es útil cuando conoces los argumentos de la función de antemano.

Ejemplo: Reutilizar el método saludar de otro objeto.

```javascript
const persona1 = { nombre: "Ana" };
const persona2 = { nombre: "Luis" };

function saludar(ciudad, pais) {
  console.log(`Hola, soy ${this.nombre} y vivo en ${ciudad}, ${pais}.`);
}

saludar.call(persona1, "Madrid", "España"); // "Hola, soy Ana y vivo en Madrid, España."
saludar.call(persona2, "Lima", "Perú"); // "Hola, soy Luis y vivo en Lima, Perú."
```

En este ejemplo, `call` le dice a la función `saludar` que su `this` debe ser persona1 (o persona2) y que los argumentos son 'Madrid' y 'España'.

### apply(thisArg, [argsArray])

Invoca una función con un valor de `this` específico y un array o un objeto tipo array de argumentos.

Es ideal cuando tienes los argumentos en una colección o cuando no sabes cuántos argumentos vas a pasar.

Ejemplo: Encontrar el número máximo en un array.

```javascript
const numeros = [5, 2, 8, 1, 9];

// Math.max normalmente no acepta un array, pero podemos usar apply para que lo haga
const maximo = Math.max.apply(null, numeros);
console.log(maximo); // 9
```

En este caso, `apply` le pasa el array numeros como una lista de argumentos a la función `Math.max`. El valor de `this` es `null` porque no es relevante para `Math.max`.

## Resumen

- `this`: El contexto de ejecución de una función, que cambia según cómo se invoca.

- `call`: Invoca una función, define su `this` y pasa los argumentos de forma individual.

- `apply`: Invoca una función, define su `this` y pasa los argumentos en un array.
