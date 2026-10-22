<h1 align='center'>Funciones de Primera Clase</h1>

<h2>📑 Contenido</h2>

- [Funciones de Primera Clase](#funciones-de-primera-clase)
- [¿Qué significa que las funciones sean de "primera clase"?](#qué-significa-que-las-funciones-sean-de-primera-clase)
- [Asignación de funciones a variables](#asignación-de-funciones-a-variables)
- [Pasar funciones como argumentos](#pasar-funciones-como-argumentos)
- [Devolver funciones desde otras funciones](#devolver-funciones-desde-otras-funciones)
- [Almacenar funciones en estructuras de datos](#almacenar-funciones-en-estructuras-de-datos)
- [Funciones de orden superior](#funciones-de-orden-superior)
- [Clousures y funciones de primera clase](#clousures-y-funciones-de-primera-clase)

## Funciones de Primera Clase

En JavaScript, las funciones son tratadas como ciudadanos de primera clase o entidades de primera clase ("first-class citizens"). Esto significa que las funciones en JavaScript tienen las mismas propiedades y capacidades que otros valores, como los números o las cadenas de texto. Pueden ser asignadas a variables, pasadas como argumentos a otras funciones y devueltas por otras funciones. Este concepto es clave para entender la flexibilidad y el poder de JavaScript, especialmente en la programación funcional.

## ¿Qué significa que las funciones sean de "primera clase"?

Que las funciones en JavaScript sean de "primera clase" significa que son tratadas como cualquier otro valor en el lenguaje. Es decir, pueden:

- Ser asignadas a variables.
- Ser pasadas como argumentos a otras funciones.
- Ser devueltas por otras funciones.
- Ser almacenadas en estructuras de datos, como arrays u objetos.

En muchos lenguajes de programación, las funciones no se tratan como un tipo de dato regular, pero en JavaScript, son objetos de pleno derecho. Esto abre la puerta a una gran cantidad de patrones y estilos de programación, como las funciones de orden superior y la programación funcional.

## Asignación de funciones a variables

En JavaScript, puedes asignar una función a una variable de la misma manera que asignarías un valor como un número o una cadena de texto.

```js
const saludar = function () {
  console.log("¡Hola!");
};

saludar(); // Imprime: ¡Hola!
```

En este ejemplo, la función anónima que imprime "¡Hola!" se asigna a la variable saludar. Posteriormente, saludar puede ser llamada como si fuera una función nombrada.

## Pasar funciones como argumentos

Una de las características más poderosas de las funciones de primera clase es que pueden ser pasadas como argumentos a otras funciones. Esto es muy común en JavaScript, especialmente cuando se utilizan callbacks.

```js
function ejecutarFuncion(funcion) {
  funcion(); // Ejecuta la función que se pasa como argumento
}

ejecutarFuncion(function () {
  console.log("Esta es una función pasada como argumento");
});
```

Aquí, ejecutarFuncion acepta una función como argumento y la ejecuta dentro de su cuerpo. Este patrón es fundamental en JavaScript, ya que permite crear funciones reutilizables que dependen del comportamiento definido por otras funcione

## Devolver funciones desde otras funciones

JavaScript también permite que una función devuelva otra función. Este es un concepto crucial en la creación de closures y otros patrones avanzados de programación funcional.

```js
function crearMultiplicador(factor) {
  return function (numero) {
    return numero * factor;
  };
}

const duplicar = crearMultiplicador(2);
console.log(duplicar(5)); // Imprime: 10

const triplicar = crearMultiplicador(3);
console.log(triplicar(5)); // Imprime: 15
```

En este ejemplo, crearMultiplicador devuelve una nueva función que multiplica un número por el factor dado. Este patrón es útil cuando necesitas generar funciones personalizadas en tiempo de ejecución.

## Almacenar funciones en estructuras de datos

Dado que las funciones son objetos, también puedes almacenarlas en estructuras de datos como arrays u objetos, lo que permite construir colecciones de comportamientos que se pueden ejecutar dinámicamente.

```js
const funciones = [
  function () {
    console.log("Primera función");
  },
  function () {
    console.log("Segunda función");
  },
  function () {
    console.log("Tercera función");
  },
];

funciones.forEach((funcion) => funcion());
```

En este caso, almacenamos tres funciones en un array y las ejecutamos usando `forEach`. Este enfoque permite aplicar diferentes comportamientos de forma dinámica.

```js
const acciones = {
  saludar: function () {
    console.log("¡Hola!");
  },
  despedir: function () {
    console.log("¡Adiós!");
  },
};

acciones.saludar(); // Imprime: ¡Hola!
acciones.despedir(); // Imprime: ¡Adiós!
```

## Funciones de orden superior

Las funciones de orden superior son funciones que aceptan otras funciones como parámetros o devuelven funciones. Estas funciones son un aspecto central de la programación funcional en JavaScript.

```js
function aplicarOperacion(a, b, operacion) {
  return operacion(a, b);
}

const sumar = function (x, y) {
  return x + y;
};

const resultado = aplicarOperacion(5, 3, sumar);
console.log(resultado); // Imprime: 8
```

En este ejemplo, aplicarOperacion es una función de orden superior que acepta otra función (sumar) como argumento para realizar una operación sobre los números a y b.

## Clousures y funciones de primera clase

El concepto de closures está intrínsecamente relacionado con las funciones de primera clase. Un closure es una función que "recuerda" el ámbito en el que fue creada, incluso después de que ese ámbito haya desaparecido. Esto es posible gracias a que las funciones en JavaScript son de primera clase y pueden capturar el entorno léxico en el que fueron definidas.

```js
function crearContador() {
  let contador = 0;
  return function () {
    contador++;
    return contador;
  };
}

const contador1 = crearContador();
console.log(contador1()); // Imprime: 1
console.log(contador1()); // Imprime: 2

const contador2 = crearContador();
console.log(contador2()); // Imprime: 1
```

En este ejemplo, la función devuelta por crearContador sigue accediendo a la variable contador incluso después de que la ejecución de crearContador haya terminado. Esto demuestra cómo los closures permiten a las funciones recordar y acceder a su entorno léxico.
