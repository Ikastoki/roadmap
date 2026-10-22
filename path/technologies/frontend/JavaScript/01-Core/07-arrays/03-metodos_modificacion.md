<h1 align='center'>Métodos de modificación</h1>

<h2>📑 Contenido</h2>

- [Métodos de modificación](#métodos-de-modificación)
- [push()](#push)
- [pop()](#pop)
- [shift()](#shift)
- [unshift()](#unshift)
- [splice()](#splice)
- [fill()](#fill)
- [reverse()](#reverse)
- [sort()](#sort)
- [copyWithin()](#copywithin)
- [flat()](#flat)
- [flatMap()](#flatmap)

## Métodos de modificación

Los métodos de modificación de arrays permiten alterar directamente el contenido de un array, ya sea agregando, eliminando o reordenando elementos. A continuación se describen algunos de los métodos más importantes que modifican arrays en JavaScript.

## push()

El método `push()` agrega uno o más elementos al final de un array y devuelve la nueva longitud del array

```js
const arr = [1, 2, 3];
arr.push(4, 5);
console.log(arr); // [1, 2, 3, 4, 5]
```

## pop()

`pop() `elimina el último elemento de un array y devuelve dicho elemento. Si el array está vacío, devuelve undefined.

```js
const arr = [1, 2, 3];
const ultimo = arr.pop();
console.log(ultimo); // 3
console.log(arr); // [1, 2]
```

## shift()

El método `shift()` elimina el primer elemento de un array y devuelve ese elemento. Modifica la longitud del array.

```js
const arr = [1, 2, 3];
const primero = arr.shift();
console.log(primero); // 1
console.log(arr); // [2, 3]
```

## unshift()

`unshift()` agrega uno o más elementos al principio de un array y devuelve la nueva longitud del array.

```js
const arr = [2, 3];
arr.unshift(0, 1);
console.log(arr); // [0, 1, 2, 3]
```

## splice()

El método `splice()` cambia el contenido de un array eliminando, reemplazando o agregando elementos. Puede devolver los elementos eliminados.

```js
const arr = [1, 2, 3, 4, 5];

// Eliminar 2 elementos desde el índice 1
arr.splice(1, 2);
console.log(arr); // [1, 4, 5]

// Insertar elementos
arr.splice(2, 0, 6, 7);
console.log(arr); // [1, 4, 6, 7, 5]
```

## fill()

`fill()` rellena todos los elementos de un array desde un índice inicial hasta uno final con un valor estático.

```js
const arr = [1, 2, 3, 4];
arr.fill(0, 1, 3); // Rellena con 0 desde el índice 1 al 2
console.log(arr); // [1, 0, 0, 4]
```

## reverse()

reverse() invierte el orden de los elementos de un array. Modifica el array original y lo devuelve.

```js
const arr = [1, 2, 3];
arr.reverse();
console.log(arr); // [3, 2, 1]
```

## sort()

El método `sort()` ordena los elementos de un array en su lugar. Por defecto, ordena como cadenas de texto. Si se necesita un orden numérico o personalizado, debes proporcionar una función de comparación.

```js
const arr = [3, 1, 4, 2];
arr.sort(); // Ordena como texto, por lo que 1, 2, 3, 4 funciona aquí
console.log(arr); // [1, 2, 3, 4]

// Ordenar numéricamente de manera descendente
arr.sort((a, b) => b - a);
console.log(arr); // [4, 3, 2, 1]
```

## copyWithin()

El método `copyWithin()` copia una sección del array dentro de la misma matriz, sin modificar su longitud. Sobrescribe los elementos existentes en la ubicación de destino.

```js
const arr = [1, 2, 3, 4, 5];
arr.copyWithin(0, 3); // Copia los elementos desde el índice 3 al índice 0
console.log(arr); // [4, 5, 3, 4, 5]
```

## flat()

El método `flat()` crea un nuevo array con todos los elementos sub-arrays aplanados hasta la profundidad especificada. Por defecto aplana un nivel.

```js
const arr = [1, [2, 3], [[4, 5]]];
const nuevoArr = arr.flat(2); // Aplana hasta una profundidad de 2
console.log(nuevoArr); // [1, 2, 3, 4, 5]
```

## flatMap()

`flatMap()` combina las operaciones de `map()` y `flat()`. Primero mapea cada elemento usando una función de callback y luego aplana el resultado en un nuevo array.

```js
const arr = [1, 2, 3];
const nuevoArr = arr.flatMap((x) => [x, x * 2]);
console.log(nuevoArr); // [1, 2, 2, 4, 3, 6]
```
