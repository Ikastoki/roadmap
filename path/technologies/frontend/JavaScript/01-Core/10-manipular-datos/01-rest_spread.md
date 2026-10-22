<h1 align='center'>Rest y Spread </h1>

<h2>📑 Contenido</h2>

- [Rest y Spread](#rest-y-spread)
- [Operador Rest (...)](#operador-rest-)
  - [En funciones con un número variable de parámetros](#en-funciones-con-un-número-variable-de-parámetros)
  - [En la desestructuración de arrays](#en-la-desestructuración-de-arrays)
  - [En la desestructuración de objetos](#en-la-desestructuración-de-objetos)
- [Operador Spread (...)](#operador-spread-)
  - [Copiar arrays](#copiar-arrays)
  - [Combinar arrays](#combinar-arrays)
  - [Pasar argumentos a funciones](#pasar-argumentos-a-funciones)
  - [Copiar objetos](#copiar-objetos)
  - [Combinar objetos](#combinar-objetos)

## Rest y Spread

En JavaScript, las operadores Rest (`...`) y Spread (`...`) son dos herramientas muy poderosas que comparten la misma sintaxis de tres puntos (`...`), pero se usan en contextos diferentes y con propósitos distintos. Ambos operadores permiten escribir código más conciso y claro, especialmente cuando trabajas con arrays, objetos y funciones que aceptan un número variable de argumentos.

## Operador Rest (...)

El operador Rest (`...`) permite agrupar el resto de los elementos en una función o en una asignación en un array o un objeto. En otras palabras, recoge múltiples valores en una única variable como un array o un objeto.

```js
function(param1, param2, ...rest) {
  // Código de la función
}
```

### En funciones con un número variable de parámetros

Cuando no sabes cuántos argumentos va a recibir una función, el operador Rest te permite agrupar todos los argumentos adicionales en un array.

```js
function sumar(...numeros) {
  return numeros.reduce((a, b) => a + b, 0);
}

console.log(sumar(1, 2, 3)); // 6
console.log(sumar(4, 5)); // 9
```

En este ejemplo, numeros captura todos los argumentos pasados a la función como un array y se suman usando el método `reduce()`.

### En la desestructuración de arrays

El operador Rest también se puede usar en la desestructuración de arrays para capturar los elementos restantes.

```js
let [a, b, ...resto] = [1, 2, 3, 4, 5];
console.log(a); // 1
console.log(b); // 2
console.log(resto); // [3, 4, 5]
```

Aquí, los primeros dos valores del array se asignan a a y b, y el resto de los valores se agrupan en el array resto.

### En la desestructuración de objetos

De manera similar, puedes usar el operador Rest en la desestructuración de objetos para agrupar las propiedades restantes en un nuevo objeto.

```js
let { nombre, ...datosRestantes } = {
  nombre: "Juan",
  edad: 30,
  ciudad: "Madrid",
};
console.log(nombre); // "Juan"
console.log(datosRestantes); // { edad: 30, ciudad: "Madrid" }
```

En este ejemplo, el operador Rest agrupa las propiedades edad y ciudad en el objeto datosRestantes.

## Operador Spread (...)

El operador Spread (`...`) se utiliza para expandir un array o un objeto en elementos individuales. Es útil cuando necesitas pasar elementos de un array como argumentos en una función, combinar arrays, o clonar objetos de manera superficial.

```js
let nuevoArray = [...array];
let nuevoObjeto = { ...objeto };
```

### Copiar arrays

El operador Spread permite hacer una copia superficial de un array. Esto es útil cuando quieres duplicar un array sin modificar el original.

```js
let original = [1, 2, 3];
let copia = [...original];
copia.push(4);

console.log(original); // [1, 2, 3]
console.log(copia); // [1, 2, 3, 4]
```

En este ejemplo, copia es una copia independiente de original, por lo que modificar copia no afecta a original.

### Combinar arrays

Puedes usar el operador Spread para combinar múltiples arrays en uno solo.

```js
let array1 = [1, 2];
let array2 = [3, 4];
let combinado = [...array1, ...array2];

console.log(combinado); // [1, 2, 3, 4]
```

Aquí, los elementos de array1 y array2 se expanden y se combinan en un solo array combinado.

### Pasar argumentos a funciones

El operador Spread es muy útil cuando tienes un array de datos que quieres pasar como argumentos individuales a una función.

```js
function sumar(a, b, c) {
  return a + b + c;
}

let numeros = [1, 2, 3];
console.log(sumar(...numeros)); // 6
```

En este caso, el operador Spread expande el array numeros en tres argumentos individuales.

### Copiar objetos

El operador Spread también puede copiar objetos de manera superficial.

```js
let objetoOriginal = { nombre: "Ana", edad: 25 };
let copiaObjeto = { ...objetoOriginal };

console.log(copiaObjeto); // { nombre: "Ana", edad: 25 }
```

Al igual que con los arrays, la copia es superficial, lo que significa que si el objeto tiene propiedades que son referencias (como arrays u otros objetos), esas referencias no se copiarán profundamente.

### Combinar objetos

El operador Spread es muy útil cuando quieres combinar propiedades de varios objetos en uno solo.

```js
let objeto1 = { nombre: "Ana", edad: 25 };
let objeto2 = { ciudad: "Madrid", edad: 30 };
let combinado = { ...objeto1, ...objeto2 };

console.log(combinado); // { nombre: "Ana", edad: 30, ciudad: "Madrid" }
```

Si hay propiedades duplicadas (como edad en este ejemplo), las últimas sobrescribirán las anteriores.
