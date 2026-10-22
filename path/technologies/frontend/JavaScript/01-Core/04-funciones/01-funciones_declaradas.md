<h1 align='center'>Funciones</h1>

<h2>📑 Contenido</h2>

- [Funciones](#funciones)
- [Funciones Declaradas](#funciones-declaradas)
- [Definición e Invocación](#definición-e-invocación)
- [Parámetros y Argumentos](#parámetros-y-argumentos)
- [Parámetros Predeterminados (Default Parameters)](#parámetros-predeterminados-default-parameters)
- [El objeto arguments](#el-objeto-arguments)
- [El valor de retorno](#el-valor-de-retorno)

## Funciones

Las funciones son uno de los conceptos más importantes en cualquier lenguaje de programación, y en JavaScript juegan un rol fundamental. Las funciones permiten organizar el código, evitar la repetición y dividir las tareas en bloques lógicos que pueden reutilizarse.

## Funciones Declaradas

Las funciones declaradas (o funciones definidas con la palabra clave function) son una de las formas más comunes de definir funciones en JavaScript. Son características fundamentales del lenguaje y tienen varias propiedades que las hacen únicas.

## Definición e Invocación

Una función en JavaScript es un bloque de código diseñado para realizar una tarea específica. Las funciones se definen una vez y se pueden reutilizar en cualquier parte del código.

Para invocar (o llamar) a la función, simplemente se utiliza su nombre seguido de paréntesis.

```js
// Se define la función
function saludar() {
  console.log("¡Hola, mundo!");
}

saludar(); // Invoca la función, imprime "¡Hola, mundo!"
```

En este ejemplo, la función saludar es definida y luego invocada. Cada vez que la llamemos, ejecutará el código dentro de su cuerpo.

## Parámetros y Argumentos

Las funciones en JavaScript pueden recibir parámetros. Los parámetros son valores que se pasan a la función para personalizar su comportamiento. Al momento de invocar la función, los valores específicos que le pasamos se denominan argumentos.

```js
// Parámetros (a y b)
function sumar(a, b) {
  return a + b;
}

let resultado = sumar(5, 3); // Llamada con argumentos 5 y 3
console.log(resultado); // Imprime 8
```

En este ejemplo, sumar recibe dos parámetros (a y b) que son utilizados dentro de la función. Cuando llamamos a sumar(5, 3), estamos pasando los argumentos 5 y 3, que serán sumados dentro de la función.

## Parámetros Predeterminados (Default Parameters)

En JavaScript, puedes definir valores predeterminados para los parámetros de una función. Si no se pasa un valor para ese parámetro cuando se llama a la función, el parámetro tomará el valor predeterminado.

```js
function saludar(nombre = "Amigo") {
  return "Hola, " + nombre;
}

console.log(saludar()); // Imprime "Hola, Amigo"
console.log(saludar("María")); // Imprime "Hola, María"
```

Aquí, si no se proporciona un nombre, la función utiliza el valor "Amigo" como predeterminado.

## El objeto arguments

Dentro de cualquier función "regular" (no en funciones de flecha), existe un objeto llamado `arguments` que contiene todos los argumentos pasados a la función, independientemente de cuántos parámetros se hayan definido en la declaración de la función. Esto permite que una función maneje un número indefinido de argumentos.

> [!WARNING]
> El objeto arguments no es en las funciones flechas. Esto significa que, si intentas usar arguments dentro de una función flecha, obtendrás un `ReferenceError`.

```js
function sumarTodos() {
  let suma = 0;
  for (let i = 0; i < arguments.length; i++) {
    suma += arguments[i];
  }
  return suma;
}

console.log(sumarTodos(1, 2, 3, 4)); // Resultado: 10
```

En este ejemplo, la función `sumarTodos` puede sumar cualquier cantidad de argumentos utilizando el objeto `arguments`.

## El valor de retorno

Las funciones en JavaScript pueden devolver un valor utilizando la palabra clave return. Cuando se ejecuta una declaración return, la ejecución de la función se detiene y el valor especificado se devuelve al lugar donde se llamó la función.

```js
function multiplicar(a, b) {
  return a * b;
}

let producto = multiplicar(4, 5); // producto será 20
console.log("El producto es: " + producto);
```

En este caso, la función multiplicar devuelve el resultado de multiplicar los dos números. El valor devuelto es almacenado en la variable producto.

Si una función no tiene una instrucción return, devuelve undefined por defecto.
