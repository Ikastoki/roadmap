<h1 align='center'>Set</h1>

<h2>📑 Contenido</h2>

- [Set](#set)
- [Características de Set](#características-de-set)
- [Creación de un Set](#creación-de-un-set)
- [Métodos y propiedades de Set](#métodos-y-propiedades-de-set)
  - [add(valor)](#addvalor)
  - [has(valor)](#hasvalor)
  - [delete(valor)](#deletevalor)
  - [clear()](#clear)
  - [size](#size)
- [Iteración en Set](#iteración-en-set)
  - [for...of](#forof)
  - [forEach()](#foreach)
- [Casos de uso de Set](#casos-de-uso-de-set)

## Set

Un Set es una colección de valores únicos. A diferencia de los arrays, donde los valores pueden repetirse, un Set garantiza que cada valor esté presente una sola vez. Los valores pueden ser de cualquier tipo de dato, incluidos objetos.

## Características de Set

- Almacena valores únicos, no se permiten duplicados.
- Mantiene los valores en el orden en que se insertan.
- Puede almacenar cualquier tipo de dato, incluyendo objetos, funciones, números, cadenas, etc.

## Creación de un Set

Para crear un Set, se utiliza el constructor Set():

```js
let miSet = new Set();
```

También puedes inicializar un Set con un array de valores:

```js
let miSet = new Set([1, 2, 3, 3, 4, 5]);
console.log(miSet); // Set {1, 2, 3, 4, 5} - El valor 3 solo aparece una vez
```

## Métodos y propiedades de Set

### add(valor)

Añade un valor al Set. Si el valor ya está presente, no se agrega de nuevo.

```js
miSet.add(6);
```

### has(valor)

Verifica si un valor está presente en el Set.

```js
console.log(miSet.has(2)); // true
```

### delete(valor)

Elimina un valor del Set.

```js
miSet.delete(3);
```

### clear()

Elimina todos los valores del Set.

```js
miSet.clear();
```

### size

Devuelve el número de elementos en el Set.

```js
console.log(miSet.size); // 0 después de clear()
```

## Iteración en Set

Al igual que los arrays, un Set es iterable y se puede recorrer usando `for...of` o `forEach()`.

### for...of

```js
let numeros = new Set([1, 2, 3]);

for (let numero of numeros) {
  console.log(numero);
}
// 1, 2, 3
```

### forEach()

```js
numeros.forEach((valor) => {
  console.log(valor);
});
```

> [!NOTE]
> Además, los objetos Set tienen métodos adicionales como keys(), values(), y entries() que devuelven iteradores, aunque keys() y values() son esencialmente lo mismo en un Set, ya que solo hay valores, sin claves.

## Casos de uso de Set

Un Set es ideal para eliminar duplicados de arrays:

```js
let arrayDuplicado = [1, 2, 2, 3, 4, 4, 5];
let setSinDuplicados = new Set(arrayDuplicado);
console.log([...setSinDuplicados]); // [1, 2, 3, 4, 5]
```

También es útil cuando se necesita comprobar rápidamente la existencia de un valor sin preocuparse por duplicados:

```js
let palabras = new Set(["hola", "mundo", "javascript", "hola"]);
console.log(palabras.size); // 3
```
