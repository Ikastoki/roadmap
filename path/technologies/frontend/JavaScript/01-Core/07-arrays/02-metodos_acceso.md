<h1 align='center'>Métodos de acceso</h1>

<h2>📑 Contenido</h2>

- [Métodos de acceso](#métodos-de-acceso)
- [at()](#at)
- [concat()](#concat)
- [includes()](#includes)
- [indexOf()](#indexof)
- [lastIndexOf()](#lastindexof)
- [join()](#join)
- [slice()](#slice)
- [toString()](#tostring)
- [toLocaleString()](#tolocalestring)

## Métodos de acceso

Los métodos de acceso de arrays permiten obtener información de un array, acceder a sus elementos o realizar ciertas operaciones que no modifican directamente el array original. A continuación se describen algunos de los métodos más utilizados para acceder a los elementos de un array en JavaScript:

## at()

El método `at()` permite acceder a un elemento específico de un array mediante su índice. Es especialmente útil porque permite usar índices negativos para contar desde el final del array.

```js
const arr = [10, 20, 30, 40, 50];
console.log(arr.at(2)); // 30 (accede al elemento en la posición 2)
console.log(arr.at(-1)); // 50 (accede al último elemento)
```

## concat()

El método `concat()` se utiliza para combinar dos o más arrays, devolviendo un nuevo array sin modificar los originales.

```js
const arr1 = [1, 2];
const arr2 = [3, 4];
const arr3 = arr1.concat(arr2);
console.log(arr3); // [1, 2, 3, 4]
```

## includes()

Este método verifica si un array contiene un valor específico y devuelve true o false según el caso.

```js
const arr = [1, 2, 3, 4];
console.log(arr.includes(3)); // true
console.log(arr.includes(5)); // false
console.log(arr.includes(2, 2)); // false (comienza la búsqueda desde el índice 2)
```

## indexOf()

`indexOf()` devuelve el primer índice en el que se encuentra un elemento dentro del array. Si el elemento no se encuentra, devuelve -1.

```js
const arr = [10, 20, 30, 10];
console.log(arr.indexOf(10)); // 0 (primer índice de 10)
console.log(arr.indexOf(30)); // 2
console.log(arr.indexOf(50)); // -1 (no encontrado)
```

## lastIndexOf()

Es similar a `indexOf()`, pero busca el último índice donde aparece un valor en el array.

```js
const arr = [10, 20, 30, 10];
console.log(arr.lastIndexOf(10)); // 3 (último índice de 10)
console.log(arr.lastIndexOf(20)); // 1
console.log(arr.lastIndexOf(50)); // -1 (no encontrado)
```

## join()

El método `join()` convierte todos los elementos de un array en una cadena, separándolos por el delimitador especificado. Si no se proporciona un delimitador, usa una coma por defecto.

```js
const arr = ["JavaScript", "es", "genial"];
console.log(arr.join(" ")); // "JavaScript es genial"
console.log(arr.join("-")); // "JavaScript-es-genial"
```

## slice()

`slice()` devuelve una porción de un array sin modificar el array original. Especificas un índice de inicio y, opcionalmente, un índice de fin.

```js
const arr = [1, 2, 3, 4, 5];
console.log(arr.slice(1, 3)); // [2, 3] (elementos entre índice 1 y 3)
console.log(arr.slice(2)); // [3, 4, 5] (desde el índice 2 hasta el final)
```

## toString()

El método `toString()` convierte un array en una cadena de texto. Cada elemento del array se convierte en una cadena, y se separa por comas.

```js
const arr = [1, 2, "a", "b"];
console.log(arr.toString()); // "1,2,a,b"
```

## toLocaleString()

Este método convierte los elementos de un array a cadenas usando las convenciones locales para cada tipo de dato (como fechas o números). Es útil cuando quieres formatear elementos según la configuración local del navegador.

```js
const fecha = new Date();
const numeros = [12345.6789, fecha];
console.log(numeros.toLocaleString("es-ES"));
// "12.345,679, [fecha en formato local]"
```
