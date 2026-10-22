<h1 align='center'>Cuantificadores</h1>

<h2>📑 Contenido</h2>

- [Cuantificadores](#cuantificadores)
- [Cuantificador Exacto (`{n}`)](#cuantificador-exacto-n)
- [Cuantificador de Rango (`{n,m}`)](#cuantificador-de-rango-nm)
- [Cuantificador de Mínimo ({n,})](#cuantificador-de-mínimo-n)
- [Cuantificador Opcional (`?`)](#cuantificador-opcional-)
- [Cuantificador de Uno o Más (`+`)](#cuantificador-de-uno-o-más-)
- [Cuantificador de Cero o Más (`*`)](#cuantificador-de-cero-o-más-)
- [Cuantificadores Codiciosos y Perezosos](#cuantificadores-codiciosos-y-perezosos)
  - [Cuantificadores Perezosos (`*?, +?, {n,m}?`)](#cuantificadores-perezosos---nm)

## Cuantificadores

Los cuantificadores en las expresiones regulares son elementos que controlan la cantidad de veces que un carácter, un grupo o una clase de caracteres debe aparecer para que se considere una coincidencia. Estos permiten especificar si queremos que un patrón aparezca exactamente un número de veces, al menos una vez, o dentro de un rango específico de repeticiones.

## Cuantificador Exacto (`{n}`)

Este cuantificador indica que el patrón anterior debe aparecer exactamente n veces.

```js
let regex = /a{3}/; // Coincide con exactamente tres letras "a" seguidas
console.log(regex.test("aa")); // false (solo hay dos "a")
console.log(regex.test("aaa")); // true (hay tres "a")
console.log(regex.test("aaaa")); // true (hay más de tres "a", pero coincide con las primeras tres)
```

En este caso, `/a{3}/` coincide solo cuando hay exactamente tres letras "a" seguidas.

## Cuantificador de Rango (`{n,m}`)

El cuantificador de rango permite definir un número mínimo y máximo de repeticiones. El patrón debe aparecer al menos n veces y como máximo m veces.

```js
let regex = /a{2,4}/; // Coincide con entre dos y cuatro letras "a"
console.log(regex.test("a")); // false (solo hay una "a")
console.log(regex.test("aa")); // true (hay dos "a")
console.log(regex.test("aaaa")); // true (hay cuatro "a")
console.log(regex.test("aaaaa")); // true (coincide con las primeras cuatro "a")
```

Aquí, `/a{2,4}/` coincide si la letra "a" aparece entre 2 y 4 veces seguidas.

## Cuantificador de Mínimo ({n,})

Este cuantificador indica que el patrón debe aparecer al menos n veces, pero no tiene un máximo. Puede repetirse indefinidamente.

```js
let regex = /a{2,}/; // Coincide con al menos dos letras "a"
console.log(regex.test("a")); // false (solo hay una "a")
console.log(regex.test("aa")); // true (hay dos "a")
console.log(regex.test("aaaaa")); // true (hay cinco "a", suficiente para el mínimo de dos)
```

En este caso, `/a{2,}/` busca al menos dos letras "a", pero no establece un límite superior para el número de repeticiones.

## Cuantificador Opcional (`?`)

El cuantificador `?` indica que el patrón anterior debe aparecer cero o una vez. Es decir, es opcional, pero si aparece, solo debe aparecer una vez.

```js
let regex = /colou?r/; // Coincide con "color" o "colour"
console.log(regex.test("color")); // true (sin "u")
console.log(regex.test("colour")); // true (con "u")
console.log(regex.test("colouur")); // false (más de una "u")
```

El patrón `/colou?r/` coincidirá tanto con la palabra "color" como con "colour" porque la "u" es opcional.

## Cuantificador de Uno o Más (`+`)

El cuantificador `+` indica que el patrón debe aparecer al menos una vez, pero puede repetirse indefinidamente.

```js
let regex = /a+/; // Coincide con una o más letras "a"
console.log(regex.test("b")); // false (no hay "a")
console.log(regex.test("a")); // true (hay una "a")
console.log(regex.test("aaaa")); // true (hay cuatro "a")
```

En este caso, `/a+/` busca al menos una letra "a", pero también coincidirá si hay más.

## Cuantificador de Cero o Más (`*`)

El cuantificador `*` indica que el patrón debe aparecer cero o más veces. Esto significa que el patrón es opcional, pero si aparece, puede hacerlo cualquier cantidad de veces.

```js
let regex = /a*/; // Coincide con cero o más letras "a"
console.log(regex.test("")); // true (no hay "a", pero el patrón puede aparecer cero veces)
console.log(regex.test("a")); // true (hay una "a")
console.log(regex.test("aaaaa")); // true (hay cinco "a")
```

El patrón `/a*/` coincidirá incluso si no hay letras "a" presentes, ya que la expresión regular permite que aparezca cero veces.

## Cuantificadores Codiciosos y Perezosos

Por defecto, los cuantificadores como `*, +, y {n,m}` son codiciosos, lo que significa que intentan capturar la mayor cantidad posible de caracteres que coincidan con el patrón.

```js
let regex = /a.+b/; // Coincide con "a" seguido de cualquier número de caracteres y una "b"
console.log("acbdab".match(regex)); // ["acbdab"] (captura la mayor cantidad posible entre "a" y "b")
```

El patrón `/a.+b/` coincide con la primera "a" y la última "b", capturando todo lo que hay en medio.

### Cuantificadores Perezosos (`*?, +?, {n,m}?`)

Para hacer que un cuantificador sea perezoso, se agrega un signo de interrogación ? después del cuantificador. Los cuantificadores perezosos intentan capturar la menor cantidad posible de caracteres.

```js
let regex = /a.+?b/; // Coincide con "a" seguido de la menor cantidad de caracteres y una "b"
console.log("acbdab".match(regex)); // ["acb"] (captura lo mínimo entre la primera "a" y "b")
```

El patrón `/a.+?b/` captura solo lo necesario para cumplir con la expresión, en lugar de capturar todo lo que pueda.
