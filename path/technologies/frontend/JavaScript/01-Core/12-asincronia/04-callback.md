<h1 align='center'>Callback </h1>

<h2>📑 Contenido</h2>

- [Callback](#callback)
- [¿Qué es un Callback?](#qué-es-un-callback)
- [Callbacks en Operaciones Asíncronas](#callbacks-en-operaciones-asíncronas)
- [Callback en Funciones Nativas](#callback-en-funciones-nativas)
  - [Ejemplo con forEach](#ejemplo-con-foreach)
  - [Ejemplo con map](#ejemplo-con-map)
- [Funciones Anónimas y Callbacks](#funciones-anónimas-y-callbacks)
- [Callback Hell](#callback-hell)
  - [Solución al Callback Hell](#solución-al-callback-hell)
- [Manejo de Errores con Callbacks](#manejo-de-errores-con-callbacks)
- [Callbacks vs. Promesas y async/await](#callbacks-vs-promesas-y-asyncawait)

## Callback

Un callback es una función que se pasa como argumento a otra función y se ejecuta después de que se haya completado algún tipo de operación. Este concepto es fundamental en JavaScript, especialmente porque el lenguaje está diseñado para ejecutarse de manera asíncrona y no bloqueante. Los callbacks permiten manejar el flujo de ejecución cuando se necesita esperar a que algo suceda, como una respuesta de un servidor, la lectura de un archivo, o un temporizador.

## ¿Qué es un Callback?

Un callback es simplemente una función que se pasa a otra función como argumento. La función a la que se le pasa el callback decide cuándo y cómo ejecutarlo. A menudo, los callbacks se utilizan para manejar resultados de operaciones asíncronas, es decir, tareas que no se completan de manera inmediata.

```js
function saludo(nombre) {
  console.log(`Hola, ${nombre}`);
}

function procesarEntradaUsuario(callback) {
  const nombre = "Ana";
  callback(nombre);
}

procesarEntradaUsuario(saludo); // "Hola, Ana"
```

En este ejemplo:

- saludo es la función que actúa como callback.
- procesarEntradaUsuario es una función que recibe un callback como argumento y lo ejecuta pasando el nombre como parámetro.

## Callbacks en Operaciones Asíncronas

Las operaciones asíncronas, como la lectura de archivos, las solicitudes de red, o los temporizadores, no pueden ejecutarse de manera inmediata. Los callbacks permiten especificar qué hacer una vez que esa operación ha terminado.

Ejemplo de Callback Asíncrono con setTimeout

```js
function mostrarMensaje() {
  console.log("Este mensaje aparece después de 3 segundos");
}

setTimeout(mostrarMensaje, 3000);
```

Aquí, el callback es la función mostrarMensaje, que será ejecutada después de 3 segundos por el `setTimeout`. El código no se detiene mientras se espera, sino que JavaScript continúa ejecutando el resto del código, y cuando pasan los 3 segundos, ejecuta el callback.

## Callback en Funciones Nativas

Algunas funciones nativas de JavaScript, como `forEach, map, o filter`, también utilizan callbacks. Estas funciones permiten ejecutar una operación en cada elemento de un array de manera muy eficiente.

### Ejemplo con forEach

```js
const numeros = [1, 2, 3, 4, 5];

numeros.forEach(function (numero) {
  console.log(numero * 2);
});
```

- `forEach` es un método del array que toma una función de callback.
- El callback recibe cada elemento del array (en este caso, los números) y se ejecuta sobre cada uno de ellos.

### Ejemplo con map

```js
const numeros = [1, 2, 3, 4, 5];
const dobles = numeros.map(function (numero) {
  return numero * 2;
});
console.log(dobles); // [2, 4, 6, 8, 10]
```

En este caso, la función `map` utiliza un callback para modificar cada valor del array original, devolviendo un nuevo array con los resultados.

## Funciones Anónimas y Callbacks

Muchas veces, en lugar de pasar una función definida por su nombre, es común utilizar funciones anónimas como callbacks. Las funciones anónimas son aquellas que no tienen un nombre y se definen directamente donde se necesitan.

```js
setTimeout(function () {
  console.log("Este mensaje aparece después de 2 segundos");
}, 2000);
```

Aquí, el callback es una función anónima que imprime un mensaje después de 2 segundos. No necesita estar definida por separado, lo que puede hacer el código más compacto.

## Callback Hell

Aunque los callbacks son muy útiles, si no se estructuran correctamente, pueden llevar a lo que se conoce como callback hell, una situación donde las funciones de callback se anidan unas dentro de otras, dificultando la legibilidad y el mantenimiento del código.

```js
setTimeout(() => {
  console.log("Primero");
  setTimeout(() => {
    console.log("Segundo");
    setTimeout(() => {
      console.log("Tercero");
    }, 1000);
  }, 1000);
}, 1000);
```

Este código, aunque funcional, es difícil de leer y mantener, debido al anidamiento excesivo de callbacks.

### Solución al Callback Hell

JavaScript ha introducido otras formas más modernas de manejar operaciones asíncronas, como Promesas y async/await, que mejoran la legibilidad y gestión de errores. Sin embargo, los callbacks siguen siendo un concepto importante y siguen utilizándose en muchas funciones.

## Manejo de Errores con Callbacks

Un problema con los callbacks tradicionales es la dificultad para manejar errores de manera adecuada. Muchas veces, los callbacks pueden aceptar dos parámetros: uno para el error (si ocurre) y otro para el resultado exitoso.

```js
function leerArchivo(ruta, callback) {
  // Simulación de operación de lectura de archivo
  const error = false;
  const contenido = "Contenido del archivo";

  if (error) {
    callback("Error leyendo el archivo", null);
  } else {
    callback(null, contenido);
  }
}

leerArchivo("archivo.txt", function (error, contenido) {
  if (error) {
    console.log("Ocurrió un error: " + error);
  } else {
    console.log("Archivo leído: " + contenido);
  }
});
```

- El callback tiene dos argumentos: el primero es el error (si ocurre), y el segundo es el resultado exitoso.
- Si ocurre un error, se maneja dentro de la función callback.

## Callbacks vs. Promesas y async/await

Aunque los `callbacks` son una forma efectiva de manejar la asincronía, tienen sus limitaciones, como el callback hell o la dificultad para manejar errores y control de flujo.

`Promesas` y `async/await` son mecanismos más modernos que mejoran la sintaxis y manejabilidad del código asíncrono en JavaScript
