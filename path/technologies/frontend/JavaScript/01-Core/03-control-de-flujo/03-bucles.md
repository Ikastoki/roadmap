<h1 align='center'>Bucles</h1>

<h2>📑 Contenido</h2>

- [Bucles](#bucles)
- [El Bucle while](#el-bucle-while)
- [El Bucle do...while](#el-bucle-dowhile)
- [El Bucle for](#el-bucle-for)
- [El Bucle for...in](#el-bucle-forin)
- [El Bucle for...of](#el-bucle-forof)
- [Bucles anidados](#bucles-anidados)
- [Bucles infinitos](#bucles-infinitos)
- [Infinito explicito](#infinito-explicito)
  - [¿Qué pasa en for (;;) {}?](#qué-pasa-en-for--)
  - [¿Cuándo usarlo?](#cuándo-usarlo)
  - [¿Cómo detener un bucle infinito?](#cómo-detener-un-bucle-infinito)

## Bucles

En programación, un bucle (también llamado loop) es una estructura que permite repetir una o más instrucciones varias veces, mientras se cumpla una condición o hasta que se alcance un objetivo. Los bucles son fundamentales para la automatización de tareas repetitivas y para trabajar con grandes conjuntos de datos de manera eficiente.

JavaScript ofrece varios tipos de bucles que cubren diferentes casos de uso, desde repetir una tarea un número fijo de veces hasta recorrer colecciones de datos como arrays u objetos. En este capítulo exploraremos los principales tipos de bucles en JavaScript y cómo utilizarlos de manera efectiva.

## El Bucle while

El bucle `while` ejecuta un bloque de código mientras una condición especificada sea verdadera. La condición se evalúa antes de cada iteración del bucle, y si es falsa desde el principio, el bloque de código nunca se ejecuta.

```js
let contador = 0;

while (contador < 5) {
  console.log("Contador: " + contador);
  contador++;
}
```

En este ejemplo, el bucle imprimirá el valor del contador desde 0 hasta 4. En cada iteración, la variable contador se incrementa en 1, y cuando alcanza 5, el bucle se detiene porque la condición contador < 5 ya no es verdadera.

## El Bucle do...while

A diferencia del bucle `while`, en el bucle do...`while` el bloque de código se ejecuta al menos una vez antes de evaluar la condición. Luego, mientras la condición siga siendo verdadera, el bucle continuará ejecutándose.

```js
let contador = 0;

do {
  console.log("Contador: " + contador);
  contador++;
} while (contador < 5);
```

Este ejemplo produce el mismo resultado que el bucle `while`. La diferencia clave es que el bloque de código se ejecuta al menos una vez, incluso si la condición es falsa desde el principio.

## El Bucle for

El bucle `for` es uno de los más utilizados y es ideal cuando se conoce de antemano cuántas veces debe ejecutarse el bucle. Su estructura incluye tres partes clave: la inicialización, la condición y la actualización de una variable de control.

```js
for (inicialización; condición; actualización) {
  // Código que se ejecuta mientras la condición sea verdadera
}

/* 
Inicialización: Se ejecuta una vez al comienzo del bucle, generalmente para inicializar una variable de control.

Condición: Se evalúa antes de cada iteración, y si es falsa, el bucle se detiene.

Actualización: Se ejecuta después de cada iteración, generalmente para modificar la variable de control.
*/
```

Este bucle imprimirá los números del 0 al 4. La variable i se inicializa en 0, y en cada iteración se incrementa en 1. Cuando i alcanza 5, el bucle se detiene porque la condición i < 5 deja de ser verdadera.

```js
for (let i = 0; i < 5; i++) {
  console.log("i: " + i);
}
```

## El Bucle for...in

El bucle `for...in` se utiliza para recorrer las propiedades de un objeto. A diferencia de los bucles anteriores, que están pensados para trabajar con números o repeticiones, `for...in` está diseñado para iterar sobre las claves de un objeto.

```js
const persona = {
  nombre: "Juan",
  edad: 30,
  ciudad: "Madrid",
};

for (let propiedad in persona) {
  console.log(propiedad + ": " + persona[propiedad]);
}
```

Este bucle recorrerá todas las propiedades del objeto persona e imprimirá tanto las claves (nombre, edad, ciudad) como los valores asociados a esas claves (Juan, 30, Madrid).

## El Bucle for...of

El bucle `for...of` es más moderno y se utiliza para iterar sobre los valores de objetos iterables como arrays, strings, mapas y conjuntos (sets). A diferencia de `for...in`, que itera sobre las propiedades de un objeto, `for...of` se centra en los valores.

```js
let numeros = [1, 2, 3, 4, 5];

for (let numero of numeros) {
  console.log(numero);
}
```

Este bucle imprimirá cada número del array numeros (1, 2, 3, 4, 5). for...of es ideal para trabajar con arrays o cadenas de texto, ya que nos permite acceder directamente a los valores sin necesidad de un índice.

## Bucles anidados

Los bucles anidados son aquellos que contienen un bucle dentro de otro. Esto es útil cuando se trabaja con estructuras de datos complejas, como arrays bidimensionales o cuando es necesario realizar operaciones repetitivas sobre grupos de datos.

```js
for (let i = 1; i <= 3; i++) {
  for (let j = 1; j <= 3; j++) {
    console.log(`i: ${i}, j: ${j}`);
  }
}
```

En este caso, el bucle externo (i) se ejecuta tres veces, y por cada iteración del bucle externo, el bucle interno (j) también se ejecuta tres veces. Esto genera todas las combinaciones posibles de i y j.

## Bucles infinitos

Un bucle infinito ocurre cuando la condición de un bucle nunca se vuelve falsa, haciendo que el código siga repitiéndose indefinidamente. Estos bucles pueden hacer que el programa se congele o crashee, por lo que es importante asegurarse de que siempre haya una forma de salir del bucle.

```js
while (true) {
  console.log("Este es un bucle infinito");
}
```

En este ejemplo, la condición true siempre es verdadera, por lo que el bucle nunca terminará.

## Infinito explicito

La estructura for (;;) {} es un bucle infinito. Se trata de una variante del bucle for en la que se omiten todas las partes opcionales de la sintaxis (inicialización, condición y actualización), lo que provoca que el ciclo se ejecute indefinidamente, ya que no hay ninguna condición que lo detenga.

```js
for (;;) {
  console.log("Este es un bucle infinito");
}
```

Este código imprimirá "Este es un bucle infinito" indefinidamente en la consola, porque no hay una condición de salida.

### ¿Qué pasa en for (;;) {}?

- **Inicialización:** No hay ninguna, por lo que no se inicializa nada.
- **Condición:** No hay una condición explícita, lo que se interpreta como siempre `true`, causando que el bucle nunca se detenga por sí solo.
- **Actualización:** No se actualiza nada después de cada iteración.

### ¿Cuándo usarlo?

Aunque es raro que se necesite un bucle infinito explícito, for (;;) {} puede ser útil en casos donde se necesita un bucle que controle de manera manual cuándo terminar, como en servidores, sistemas en tiempo real o programas que deben ejecutarse continuamente hasta que ocurra una condición externa.

### ¿Cómo detener un bucle infinito?

Para evitar que un bucle infinito bloquee tu programa, puedes utilizar una instrucción como `break` dentro del bloque de código, que detiene el bucle en algún momento:

```js
let i = 0;
for (;;) {
  console.log(i);
  i++;
  if (i > 10) {
    break; // Detiene el bucle cuando i es mayor que 10
  }
}
```

En este ejemplo, el bucle infinito se detendrá cuando i sea mayor que 10.
