<h1 align='center'>Métodos de ordenamiento</h1>

<h2>📑 Contenido</h2>

- [Métodos de ordenamiento](#métodos-de-ordenamiento)
- [sort()](#sort)
  - [Ordenamiento de cadenas (por defecto)](#ordenamiento-de-cadenas-por-defecto)
  - [Ordenamiento de números (sin función de comparación)](#ordenamiento-de-números-sin-función-de-comparación)
  - [Ordenamiento de números (con función de comparación)](#ordenamiento-de-números-con-función-de-comparación)
  - [Ordenamiento descendente](#ordenamiento-descendente)
  - [Ordenamiento de objetos](#ordenamiento-de-objetos)
- [reverse()](#reverse)
- [copyWithin()](#copywithin)
- [fill()](#fill)

## Métodos de ordenamiento

El ordenamiento de arrays es una de las operaciones más comunes al trabajar con datos en JavaScript. El lenguaje proporciona varios métodos para ordenar, transformar y organizar los elementos de un array, siendo `sort()` el método más importante. Además, existen otros métodos que te permiten trabajar con el orden de los elementos, como `reverse()` y `copyWithin()`, que veremos en este capítulo.

## sort()

El método `sort()` ordena los elementos de un array en su lugar y devuelve el array ordenado. Por defecto, los elementos se ordenan como cadenas de texto en orden lexicográfico, lo que puede producir resultados inesperados cuando se ordenan números. Sin embargo, el método acepta una función de comparación opcional para especificar el criterio de ordenamiento.

### Ordenamiento de cadenas (por defecto)

Cuando no se pasa una función de comparación, los elementos se convierten a cadenas y se ordenan en orden lexicográfico.

```js
let frutas = ["naranja", "manzana", "banano", "mango"];
frutas.sort();
console.log(frutas); // ["banano", "manzana", "mango", "naranja"]
```

### Ordenamiento de números (sin función de comparación)

Si intentas ordenar números sin una función de comparación, el resultado puede ser inesperado debido a que los números se convierten a cadenas y se ordenan lexicográficamente.

```js
let numeros = [40, 1, 5, 200];
numeros.sort();
console.log(numeros); // [1, 200, 40, 5] -> Orden lexicográfico
```

### Ordenamiento de números (con función de comparación)

Para ordenar números correctamente, necesitas proporcionar una función de comparación.

```js
let numeros = [40, 1, 5, 200];
numeros.sort(function (a, b) {
  return a - b;
});
console.log(numeros); // [1, 5, 40, 200]
```

En este caso, la función de comparación resta a de b, lo que asegura que los números se ordenen correctamente en orden ascendente.

### Ordenamiento descendente

Para ordenar en orden descendente, simplemente invierte el valor de la resta en la función de comparación.

```js
numeros.sort(function (a, b) {
  return b - a;
});
console.log(numeros); // [200, 40, 5, 1]
```

### Ordenamiento de objetos

Cuando tienes un array de objetos, puedes usar `sort()` para ordenar en función de una propiedad específica del objeto.

```js
let personas = [
  { nombre: "Ana", edad: 28 },
  { nombre: "Carlos", edad: 22 },
  { nombre: "Juan", edad: 35 },
];

personas.sort(function (a, b) {
  return a.edad - b.edad;
});

console.log(personas);
// [{ nombre: "Carlos", edad: 22 }, { nombre: "Ana", edad: 28 }, { nombre: "Juan", edad: 35 }]
```

## reverse()

El método `reverse()` invierte el orden de los elementos en el array en su lugar, lo que significa que modifica el array original. No realiza ninguna clasificación o comparación, simplemente voltea el orden actual de los elementos.

```js
let numeros = [1, 2, 3, 4, 5];
numeros.reverse();
console.log(numeros); // [5, 4, 3, 2, 1]
```

Este método es útil cuando necesitas revertir el orden de un array después de haberlo ordenado

## copyWithin()

El método `copyWithin()` copia una sección del array a otra posición dentro del mismo array, sin modificar el tamaño de este. Este método es más relacionado con la manipulación del contenido que con el ordenamiento directo, pero puede ser útil en ciertos casos donde desees reorganizar los elementos dentro del array.

```js
let array = [1, 2, 3, 4, 5];
array.copyWithin(0, 3);
console.log(array); // [4, 5, 3, 4, 5]
```

En este ejemplo, se copian los elementos desde el índice 3 (es decir, [4, 5]) a la posición 0.

## fill()

El método `fill()` modifica un array rellenando todos los elementos desde un índice inicial hasta un índice final con un valor estático.

```js
let numeros = [1, 2, 3, 4, 5];
numeros.fill(0, 2, 4);
console.log(numeros); // [1, 2, 0, 0, 5]
```

En este caso, los elementos entre los índices 2 y 4 (sin incluir 4) se rellenan con el valor 0.
