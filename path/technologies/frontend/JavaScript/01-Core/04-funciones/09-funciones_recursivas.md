<h1 align='center'>Funciones Recursivas</h1>

<h2>📑 Contenido</h2>

- [Recursividad](#recursividad)
- [¿Qué es la Recursividad?](#qué-es-la-recursividad)
  - [Recursividad: Factorial](#recursividad-factorial)
- [Cómo Funcionan las Funciones Recursivas](#cómo-funcionan-las-funciones-recursivas)
- [Caso Base y Profundidad de Recursión](#caso-base-y-profundidad-de-recursión)
- [Problemas Clásicos Resueltos con Recursividad](#problemas-clásicos-resueltos-con-recursividad)
  - [Suma de un Array](#suma-de-un-array)
  - [Búsqueda en una Estructura Jerárquica (Árbol)](#búsqueda-en-una-estructura-jerárquica-árbol)
  - [Secuencia de Fibonacci](#secuencia-de-fibonacci)
- [Recursividad de Cola (Tail Recursion)](#recursividad-de-cola-tail-recursion)

## Recursividad

La recursividad es una técnica de programación en la que una función se llama a sí misma para resolver un problema. En lugar de usar bucles o estructuras iterativas, una función recursiva aborda el problema dividiéndolo en subproblemas más pequeños hasta llegar a una condición base o de salida, que detiene las llamadas recursivas. Este enfoque es particularmente útil para resolver problemas que pueden descomponerse de manera natural en versiones más pequeñas de sí mismos, como el cálculo de factoriales, el recorrido de estructuras de datos jerárquicas (como árboles) y algoritmos de búsqueda.

## ¿Qué es la Recursividad?

La recursividad es una técnica en la cual una función se llama a sí misma directamente o a través de otra función. Este enfoque permite que una tarea grande o compleja se resuelva de manera progresiva, abordando instancias más pequeñas del mismo problema hasta llegar a una condición base, que detiene la recursión.

En esencia, una función recursiva se compone de dos partes:

- **Caso base:** La condición que detiene la recursión.
- **Llamada recursiva:** La invocación de la función a sí misma con un conjunto reducido o modificado de parámetros, acercándose al caso base.

```js
function contarHaciaAtras(n) {
  if (n <= 0) {
    console.log("Fin de la cuenta");
    return;
  }
  console.log(n);
  contarHaciaAtras(n - 1);
}

contarHaciaAtras(5);
// Imprime: 5, 4, 3, 2, 1, "Fin de la cuenta"
```

### Recursividad: Factorial

El cálculo del factorial de un número es un ejemplo clásico que puede ser resuelto recursivamente. El factorial de un número `n` (escrito como `n!`) se define como el producto de `n` por todos los números positivos menores que él, con la condición de que el factorial de `0` es `1`.

```js
function factorial(n) {
  // Caso base
  if (n === 0) {
    return 1;
  }
  // Paso recursivo
  return n * factorial(n - 1);
}

console.log(factorial(5)); // Imprime 120 (5 * 4 * 3 * 2 * 1)
```

- El caso base es cuando n es igual a 0, en cuyo caso devuelve 1.
- El paso recursivo es la llamada a factorial(n - 1) con una versión más pequeña del problema.

## Cómo Funcionan las Funciones Recursivas

Las funciones recursivas dependen de la pila de llamadas para funcionar. Cada vez que una función se llama a sí misma, se agrega una nueva entrada en la pila. Cuando se alcanza el caso base, las llamadas recursivas comienzan a resolverse en orden inverso, eliminando las entradas de la pila y devolviendo los resultados acumulados.

Pila de Llamadas en el Ejemplo de Factorial (5!):

1. factorial(5) llama a factorial(4).
1. factorial(4) llama a factorial(3).
1. factorial(3) llama a factorial(2).
1. factorial(2) llama a factorial(1).
1. factorial(1) llama a factorial(0), que devuelve 1 (caso base).
1. Luego, el resultado de cada llamada se devuelve:
   - factorial(1) devuelve 1 \* 1 = 1
   - factorial(2) devuelve 2 \* 1 = 2
   - factorial(3) devuelve 3 \* 2 = 6
   - factorial(4) devuelve 4 \* 6 = 24
   - factorial(5) devuelve 5 \* 24 = 120

## Caso Base y Profundidad de Recursión

El caso base es el elemento más importante en una función recursiva. Si no hay un caso base, o si no se llega a él, la función seguirá llamándose indefinidamente, lo que provocará un error de desbordamiento de pila (stack overflow), debido a que la memoria destinada a la pila de llamadas se llenará.

```js
function infinito(n) {
  console.log(n);
  return infinito(n + 1); // Sin caso base
}

infinito(1); // Esto eventualmente causará un error de "desbordamiento de pila"
```

Siempre debes asegurarte de que haya una condición que detenga la recursión. Además, el tamaño del problema debe reducirse en cada paso recursivo para que el caso base se alcance en un número finito de pasos.

## Problemas Clásicos Resueltos con Recursividad

Algunos problemas son particularmente adecuados para resolverse de forma recursiva debido a su naturaleza repetitiva o jerárquica. A continuación se presentan algunos ejemplos comunes.

### Suma de un Array

La suma de los elementos de un array puede resolverse de manera recursiva dividiendo el array en una versión más pequeña del mismo problema.

```js
function sumaArray(arr) {
  // Caso base: array vacío
  if (arr.length === 0) {
    return 0;
  }
  // Paso recursivo: sumar el primer elemento y el resto del array
  return arr[0] + sumaArray(arr.slice(1));
}

console.log(sumaArray([1, 2, 3, 4])); // Imprime 10
```

### Búsqueda en una Estructura Jerárquica (Árbol)

Las estructuras de datos como los árboles son ideales para resolverse con recursión, ya que se pueden descomponer en subárboles más pequeños.

```js
let arbol = {
  valor: 1,
  hijos: [
    { valor: 2, hijos: [] },
    { valor: 3, hijos: [{ valor: 4, hijos: [] }] },
  ],
};

function recorrerArbol(nodo) {
  console.log(nodo.valor);
  nodo.hijos.forEach((hijo) => recorrerArbol(hijo));
}

recorrerArbol(arbol); // Imprime 1, 2, 3, 4
```

Este ejemplo recorre todos los nodos de un árbol utilizando recursión, donde cada subárbol se trata como una versión más pequeña del árbol original.

### Secuencia de Fibonacci

La secuencia de Fibonacci es un problema clásico que puede resolverse de forma recursiva. La secuencia se define como:

```js
function fibonacci(n) {
  if (n === 0) {
    return 0;
  } else if (n === 1) {
    return 1;
  } else {
    return fibonacci(n - 1) + fibonacci(n - 2);
  }
}

console.log(fibonacci(6)); // Imprime 8 (la secuencia es 0, 1, 1, 2, 3, 5, 8)
```

## Recursividad de Cola (Tail Recursion)

La recursividad de cola es un caso especial de recursividad en el que la llamada recursiva es la última operación que se realiza dentro de la función. Algunos lenguajes optimizan automáticamente la recursividad de cola para evitar el desbordamiento de pila, aunque JavaScript no siempre realiza esta optimización.

```js
function factorialTailRecursion(n, total = 1) {
  if (n === 0) {
    return total;
  }
  return factorialTailRecursion(n - 1, n * total);
}

console.log(factorialTailRecursion(5)); // Imprime 120
```

En este ejemplo, la llamada recursiva es la última operación en la función, lo que hace que sea un candidato para optimización de tail recursion.
