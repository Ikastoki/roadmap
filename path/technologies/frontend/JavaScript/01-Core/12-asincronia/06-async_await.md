<h1 align='center'>Async y Await</h1>

<h2>📑 Contenido</h2>

- [Async y Await](#async-y-await)
- [¿Qué es async?](#qué-es-async)
- [Casos donde async es útil](#casos-donde-async-es-útil)
- [¿Qué es await?](#qué-es-await)
- [Ventaja principal de await](#ventaja-principal-de-await)
- [Ejemplo con promesas sin async/await (usando then)](#ejemplo-con-promesas-sin-asyncawait-usando-then)
- [Manejo de errores en funciones async](#manejo-de-errores-en-funciones-async)
- [Uso de Promise.all() con async/await](#uso-de-promiseall-con-asyncawait)
- [Limitaciones y Consideraciones](#limitaciones-y-consideraciones)

## Async y Await

Async y Await son dos palabras clave introducidas en ECMAScript 2017 (ES8) que simplifican el manejo de código asíncrono en JavaScript. Estas se utilizan para trabajar de manera más intuitiva y estructurada con promesas, permitiendo escribir código asíncrono que parece código síncrono. Gracias a estas herramientas, es más fácil evitar la famosa “pirámide de la perdición” que ocurre con los callbacks y hace que el flujo de código sea más fácil de entender y depurar.

Este capítulo cubrirá los conceptos básicos y ejemplos de cómo utilizar async y await.

## ¿Qué es async?

La palabra clave async se utiliza para definir una función asíncrona. Esta función devuelve automáticamente una promesa, por lo que incluso si dentro de la función no hay promesas explícitas, JavaScript transforma el valor de retorno en una promesa resuelta. Esto facilita la escritura de funciones que realizan operaciones asíncronas, como la obtención de datos de una API o la lectura de un archivo.

```js
async function miFuncion() {
  return "Hola";
}

miFuncion().then((resultado) => {
  console.log(resultado); // "Hola"
});
```

En el ejemplo anterior, aunque la función miFuncion() parece devolver un valor directo (una cadena), realmente devuelve una promesa resuelta con ese valor.

Internamente, esto es equivalente a:

```js
function miFuncion() {
  return Promise.resolve("Hola");
}
```

## Casos donde async es útil

La principal utilidad de `async` radica en el manejo de operaciones asíncronas dentro de una función que necesita devolver una promesa. Esto es especialmente útil en combinación con la palabra clave `await`.

## ¿Qué es await?

La palabra clave `await` solo puede usarse dentro de funciones marcadas con `async`. `await` se utiliza para pausar la ejecución de la función asíncrona hasta que una promesa sea resuelta. Durante ese tiempo, el código de la función queda "en espera" sin bloquear el hilo principal.

Cuando una promesa es resuelta, el valor resultante se devuelve y puede almacenarse en una variable. Si la promesa es rechazada, `await` lanzará un error que puede capturarse usando `try/catch`

```js
async function obtenerDatos() {
  let respuesta = await fetch("https://api.ejemplo.com/datos");
  let datos = await respuesta.json();
  console.log(datos);
}

obtenerDatos();
```

- `await fetch(...)` pausa la ejecución hasta que `fetch()` resuelva su promesa.
- Una vez resuelta, `await respuesta.json()` transforma la respuesta a formato JSON.
- El flujo continúa cuando los datos JSON han sido obtenidos y procesados.

## Ventaja principal de await

El uso de `await` en lugar de las tradicionales cadenas de `then()` y `catch()` hace que el código asíncrono sea más legible y se comporte como si fuera síncrono, lo que facilita su escritura y mantenimiento.

## Ejemplo con promesas sin async/await (usando then)

```js
fetch("https://api.ejemplo.com/datos")
  .then((respuesta) => respuesta.json())
  .then((datos) => console.log(datos))
  .catch((error) => console.error("Error:", error));
```

El código equivalente con async/await:

```js
async function obtenerDatos() {
  try {
    const respuesta = await fetch("https://api.ejemplo.com/datos");
    const datos = await respuesta.json();
    console.log(datos);
  } catch (error) {
    console.error("Error:", error);
  }
}

obtenerDatos();
```

Con `async y await`, el código se vuelve mucho más sencillo y se asemeja a operaciones síncronas, haciendo que sea más claro entender el flujo.

## Manejo de errores en funciones async

Cuando utilizamos `async y await`, podemos manejar los errores de una manera más clara utilizando bloques `try/catch`. Si una promesa es rechazada, podemos capturar ese error directamente en un bloque catch, en lugar de usar `.catch()` en las promesas.

```js
async function obtenerDatos() {
  try {
    let respuesta = await fetch("https://api.ejemplo.com/datos");
    let datos = await respuesta.json();
    console.log(datos);
  } catch (error) {
    console.log("Hubo un error:", error);
  }
}

obtenerDatos();
```

Si la solicitud a `fetch` falla (por ejemplo, si no hay conexión a Internet), el código lanzará una excepción que será capturada dentro del bloque `catch`. De esta manera, podemos manejar los errores de forma más intuitiva.

## Uso de Promise.all() con async/await

`Promise.all()` es una función útil para ejecutar múltiples promesas en paralelo. Con `async/await`, podemos esperar a que todas las promesas se resuelvan antes de continuar con la ejecución del código.

```js
async function obtenerDatosMultiples() {
  try {
    const [respuesta1, respuesta2] = await Promise.all([
      fetch("https://api.ejemplo.com/datos1"),
      fetch("https://api.ejemplo.com/datos2"),
    ]);

    const datos1 = await respuesta1.json();
    const datos2 = await respuesta2.json();

    console.log(datos1, datos2);
  } catch (error) {
    console.log("Hubo un error:", error);
  }
}

obtenerDatosMultiples();
```

En este ejemplo, ambas solicitudes `fetch` se ejecutan en paralelo, lo que mejora el rendimiento. La función espera a que ambas promesas se resuelvan antes de continuar con la ejecución del código.

## Limitaciones y Consideraciones

- **Solo dentro de funciones `async`:** No puedes usar `await` fuera de una función marcada como `async`. Si lo intentas, obtendrás un error de sintaxis.

- **Manejo de errores:** Asegúrate de manejar los errores correctamente cuando trabajas con promesas, especialmente cuando esperas a múltiples operaciones asíncronas.

- **Paralelismo limitado:** Aunque `await` facilita el manejo de promesas, también puede reducir el paralelismo en algunos casos. Si esperas una promesa tras otra en una secuencia, las operaciones se ejecutarán una tras otra en lugar de en paralelo.

Ejemplo con bloqueos innecesarios:

```js
async function secuencial() {
  const datos1 = await fetch("https://api.ejemplo.com/datos1");
  const datos2 = await fetch("https://api.ejemplo.com/datos2");
  console.log(datos1, datos2);
}
```

En este caso, las dos solicitudes se ejecutan de forma secuencial, lo que puede ralentizar el proceso si no es necesario que dependan una de la otra. Para mayor rendimiento, sería mejor usar `Promise.all()` si ambas solicitudes pueden ejecutarse en paralelo.
