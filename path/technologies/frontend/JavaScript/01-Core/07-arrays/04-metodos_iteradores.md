<h1 align='center'>Métodos de iteración</h1>

<h2>📑 Contenido</h2>

- [Métodos de iteración](#métodos-de-iteración)
- [forEach()](#foreach)
- [map()](#map)
- [filter()](#filter)
- [reduce()](#reduce)
- [reduceRight()](#reduceright)
- [some()](#some)
- [every()](#every)
- [find()](#find)
- [findIndex()](#findindex)
- [findLast()](#findlast)
- [findLastIndex()](#findlastindex)
- [entries()](#entries)
- [keys()](#keys)
- [values()](#values)

## Métodos de iteración

Los métodos de iteración permiten recorrer o manipular los elementos de un array de manera eficiente. Estos métodos ofrecen una forma más declarativa de trabajar con arrays y son fundamentales para realizar operaciones comunes como transformación, filtrado, reducción y búsqueda.

## forEach()

El método `forEach()` ejecuta una función proporcionada una vez por cada elemento del array, pero no devuelve un nuevo array ni interrumpe su ejecución.

```js
const arr = [1, 2, 3];
arr.forEach((el) => console.log(el * 2)); // 2, 4, 6
```

## map()

`map()` crea un nuevo array con los resultados de aplicar una función a cada uno de los elementos del array original.

```js
const arr = [1, 2, 3];
const dobles = arr.map((el) => el * 2);
console.log(dobles); // [2, 4, 6]
```

## filter()

El método `filter()` crea un nuevo array con todos los elementos que pasen una prueba especificada en la función.

```js
const arr = [1, 2, 3, 4, 5];
const mayores = arr.filter((el) => el > 3);
console.log(mayores); // [4, 5]
```

## reduce()

`reduce()` aplica una función a un acumulador y a cada elemento del array (de izquierda a derecha) para reducirlo a un solo valor.

```js
const arr = [1, 2, 3, 4];
const suma = arr.reduce((acum, el) => acum + el, 0);
console.log(suma); // 10
```

## reduceRight()

Similar a `reduce()`, pero la iteración comienza desde el último elemento y avanza hacia el primero (de derecha a izquierda).

```js
const arr = [
  [0, 1],
  [2, 3],
  [4, 5],
];
const aplanado = arr.reduceRight((acum, el) => acum.concat(el), []);
console.log(aplanado); // [4, 5, 2, 3, 0, 1]
```

## some()

`some()` verifica si al menos un elemento en el array cumple con la condición especificada en la función de prueba. Devuelve `true` si la condición es verdadera para algún elemento.

```js
const arr = [1, 2, 3, 4];
const tienePares = arr.some((el) => el % 2 === 0);
console.log(tienePares); // true (porque hay elementos pares)
```

## every()

El método `every()` comprueba si todos los elementos del array cumplen con la condición especificada en la función de prueba. Devuelve `true` si la condición es verdadera para todos los elementos.

```js

```

## find()

`find()` devuelve el primer elemento que cumple con la condición especificada en la función de prueba. Si ningún elemento cumple, devuelve `undefined`.

```js
const arr = [2, 4, 6];
const todosPares = arr.every((el) => el % 2 === 0);
console.log(todosPares); // true
```

## findIndex()

El método `findIndex()` devuelve el índice del primer elemento que cumple con la condición en la función de prueba. Si no lo encuentra, devuelve `-1`.

```js
const arr = [1, 2, 3, 4];
const indice = arr.findIndex((el) => el > 2);
console.log(indice); // 2 (índice del primer elemento mayor que 2)
```

## findLast()

`findLast()` devuelve el último elemento que cumple con la condición especificada en la función de prueba. Si ningún elemento cumple, devuelve `undefined`.

```js
const arr = [1, 2, 3, 4];
const ultimoMayorQueDos = arr.findLast((el) => el > 2);
console.log(ultimoMayorQueDos); // 4
```

## findLastIndex()

Este método devuelve el índice del último elemento que satisface la función de prueba. Si no encuentra ningún elemento, devuelve `-1`.

```js
const arr = [1, 2, 3, 4];
const ultimoIndice = arr.findLastIndex((el) => el > 2);
console.log(ultimoIndice); // 3 (índice del último elemento mayor que 2)
```

## entries()

`entries()` devuelve un nuevo iterador de array que contiene pares clave/valor para cada índice del array. Se usa para obtener tanto el índice como el valor de los elementos al iterar.

```js
const arr = ["a", "b", "c"];
const iterador = arr.entries();
for (let [indice, valor] of iterador) {
  console.log(indice, valor); // 0 'a', 1 'b', 2 'c'
}
```

## keys()

El método `keys()` devuelve un nuevo iterador de array que contiene las claves (índices) de cada elemento del array.

```js
const arr = ["a", "b", "c"];
const claves = arr.keys();
for (let clave of claves) {
  console.log(clave); // 0, 1, 2
}
```

## values()

`values()` devuelve un nuevo iterador de array que contiene los valores de cada índice del array.

```js
const arr = ["a", "b", "c"];
const valores = arr.values();
for (let valor of valores) {
  console.log(valor); // 'a', 'b', 'c'
}
```
