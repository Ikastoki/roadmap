<h1 align='center'>Funciones Anónimas</h1>

<h2>📑 Contenido</h2>

- [Funciones Anónimas](#funciones-anónimas)
- [¿Qué es una función anónima?](#qué-es-una-función-anónima)
- [Usos comunes de las funciones anónimas](#usos-comunes-de-las-funciones-anónimas)
  - [Callbacks](#callbacks)
  - [Métodos de arrays (map, filter, forEach)](#métodos-de-arrays-map-filter-foreach)
- [Funciones anónimas y ámbitos](#funciones-anónimas-y-ámbitos)
- [Funciones anónimas y funciones flecha](#funciones-anónimas-y-funciones-flecha)

## Funciones Anónimas

Una función anónima es una función que no tiene un nombre explícito. Estas funciones se definen en el momento y se pueden asignar a variables, pasar como argumentos a otras funciones o ejecutarse inmediatamente. Son un concepto importante en JavaScript, especialmente en la programación funcional y en los patrones basados en callbacks.

## ¿Qué es una función anónima?

Una función anónima es una función que no tiene un identificador o nombre asignado. Se crean en el momento y, generalmente, se utilizan donde se requiere una función de forma temporal, como en callbacks o como argumento para otras funciones.

```js
const suma = function (a, b) {
  return a + b;
};

console.log(suma(2, 3)); // Resultado: 5
```

En este ejemplo, la función que se asigna a la variable suma es anónima, ya que no tiene un nombre explícito.

## Usos comunes de las funciones anónimas

Las funciones anónimas son muy útiles en una variedad de situaciones en JavaScript, especialmente en:

### Callbacks

Los callbacks son funciones que se pasan como argumentos a otras funciones y se ejecutan en un momento posterior. Es común utilizar funciones anónimas en este contexto para definir comportamientos específicos de manera compacta.

```js
setTimeout(function () {
  console.log("Esto ocurre después de 2 segundos");
}, 2000);
```

En este ejemplo, se utiliza una función anónima como callback para el método `setTimeout`.
La función `setTimeout` ejecutar una función o un bloque de código después de un retraso especificado.

### Métodos de arrays (map, filter, forEach)

Las funciones anónimas se utilizan frecuentemente en métodos de arrays para realizar operaciones sobre sus elementos.

```js
const numeros = [1, 2, 3, 4, 5];

const dobles = numeros.map(function (numero) {
  return numero * 2;
});

console.log(dobles); // Resultado: [2, 4, 6, 8, 10]
```

Aquí, la función anónima se pasa como argumento a `map` para transformar cada elemento del array numeros.

## Funciones anónimas y ámbitos

Las funciones anónimas, al igual que las funciones nombradas, tienen su propio ámbito léxico. Esto significa que pueden acceder a las variables que están en su entorno exterior, pero también crean un ámbito propio para las variables definidas dentro de ellas.

```js
let mensaje = "Hola desde el exterior";

const saludar = function () {
  let mensaje = "Hola desde la función";
  console.log(mensaje);
};

saludar(); // Resultado: "Hola desde la función"
console.log(mensaje); // Resultado: "Hola desde el exterior"
```

Aquí, la función anónima tiene su propio ámbito y la variable mensaje dentro de la función no afecta a la variable mensaje definida en el ámbito exterior.

## Funciones anónimas y funciones flecha

Las funciones flecha también pueden ser anónimas y, de hecho, son una forma más concisa de escribir funciones anónimas.

```js
// Ejemplo con función anónima tradicional:
const multiplicar = function (a, b) {
  return a * b;
};

// Ejemplo con función flecha anónima:
const multiplicar = (a, b) => a * b;
```
