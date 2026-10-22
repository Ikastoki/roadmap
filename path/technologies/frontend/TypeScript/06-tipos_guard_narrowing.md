<h1 align="center">Type Guards y Narrowing</h1>

<h2>📑 Contenido</h2>

- [Type Guards y Narrowing](#type-guards-y-narrowing)

## Type Guards y Narrowing

Los "Type Guards" (Guardias de Tipo) y el "Narrowing" son conceptos relacionados en TypeScript que se utilizan para realizar comprobaciones en tiempo de ejecución y reducir el tipo de una variable dentro de un bloque de código, respectivamente.

## Type Guards (Guardias de Tipo)

Los "Type Guards" son funciones o expresiones que TypeScript utiliza para determinar si un valor es de un tipo específico en tiempo de ejecución. Estos guardias permiten al compilador de TypeScript inferir tipos de manera más precisa dentro de bloques de código condicionales. Los "Type Guards" se utilizan comúnmente con tipos de unión para realizar acciones basadas en el tipo de una variable en tiempo de ejecución.

```ts
// Ejemplo de Type Guard con typeof
function esNumero(valor: any): valor is number {
  return typeof valor === "number";
}

let x: number | string = "Diez";
if (esNumero(x)) {
  console.log(x.toFixed(2)); // Se considera x como number dentro de este bloque
}
```

## Narrowing (Estrechamiento)

El Narrowing es el proceso de reducir el tipo de una variable dentro de un bloque de código, basado en alguna condición. Esto se logra a través de "Type Guards", expresiones condicionales y otros mecanismos de TypeScript que permiten al compilador inferir un tipo más específico dentro de un bloque de código.

```ts
let valor: number | string = "10";
if (typeof valor === "number") {
  // El tipo de valor se estrecha a number dentro de este bloque
  console.log(valor.toFixed(2)); // Válido
} else {
  // El tipo de valor se estrecha a string dentro de este bloque
  console.log(valor.length); // Válido
}
```

## Equality (Igualdad)

La igualdad en TypeScript se refiere a la comparación de valores para determinar si son iguales. Esto se realiza utilizando operadores como == (igualdad no estricta) o === (igualdad estricta). La igualdad estricta (===) verifica tanto el valor como el tipo de los operandos, mientras que la igualdad no estricta (==) puede realizar conversiones de tipo antes de comparar los valores.

```ts
let x: number = 10;
let y: string = "10";

console.log(x == y); // true (se realiza conversión de tipo)
console.log(x === y); // false (se compara el valor y el tipo)
```

## Truthiness (Veracidad)

La "truthiness" se refiere a cómo se evalúan los valores en contextos booleanos. En JavaScript y TypeScript, todos los valores tienen una "truthiness" asociada, lo que significa que se evalúan como verdaderos (true) o falsos (false) en contextos booleanos. Por ejemplo, los valores 0, null, undefined, NaN, '' (cadena vacía) y false se consideran "falsos" (falsy), mientras que todos los demás valores se consideran "verdaderos" (truthy).

```ts
let valor: string = "Hola";

if (valor) {
  console.log("El valor es verdadero"); // Se ejecutará porque 'valor' no es una cadena vacía
} else {
  console.log("El valor es falso");
}
```

## Type Predicates (Predicados de Tipo)

Los "Type Predicates" son funciones de guardia específicas en TypeScript que devuelven un valor de tipo booleano. Estas funciones se utilizan para comprobar si un valor es de un tipo específico en tiempo de ejecución y, al mismo tiempo, estrechar el tipo de la variable dentro de un bloque de código.

```ts
function esString(valor: any): valor is string {
  return typeof valor === "string";
}

let x: number | string = "Hola";
if (esString(x)) {
  console.log(x.toUpperCase()); // Se considera x como string dentro de este bloque
}
```
