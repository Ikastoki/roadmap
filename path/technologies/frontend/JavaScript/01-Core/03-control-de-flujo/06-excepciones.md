<h1 align='center'>Excepciones</h1>

<h2>📑 Contenido</h2>

- [Excepciones](#excepciones)
- [¿Qué es una excepción?](#qué-es-una-excepción)
- [Manejo de excepciones con try y catch](#manejo-de-excepciones-con-try-y-catch)
- [Uso de finally](#uso-de-finally)
- [Lanzar excepciones con throw](#lanzar-excepciones-con-throw)
- [Tipos de errores comunes](#tipos-de-errores-comunes)
  - [Error](#error)
  - [ReferenceError](#referenceerror)
  - [TypeError](#typeerror)
  - [SyntaxError](#syntaxerror)
  - [RangeError](#rangeerror)
- [Propagación de excepciones](#propagación-de-excepciones)
- [Buenas prácticas en el manejo de excepciones](#buenas-prácticas-en-el-manejo-de-excepciones)

## Excepciones

En la programación, las excepciones son eventos que ocurren cuando algo inesperado sucede durante la ejecución de un programa. JavaScript ofrece un mecanismo para manejar estas excepciones, lo que permite que el código responda de manera adecuada a situaciones de error en lugar de fallar abruptamente. En este capítulo, exploraremos cómo manejar excepciones en JavaScript utilizando las palabras clave `try`, `catch`, `finally`, y `throw`.

## ¿Qué es una excepción?

Una excepción ocurre cuando el motor de JavaScript encuentra un problema que no puede manejar de manera automática. Este problema puede ser, por ejemplo:

- Intentar acceder a una variable que no existe.
- Ejecutar una operación matemática inválida, como dividir por cero.
- Hacer una llamada a un servidor y que esta falle por un problema de red.

Cuando se produce una excepción y no se maneja adecuadamente, el programa se detiene abruptamente y muestra un mensaje de error en la consola. Sin embargo, JavaScript nos da la capacidad de atrapar y manejar estos errores para prevenir que afecten el flujo general del programa.

## Manejo de excepciones con try y catch

Para manejar excepciones, JavaScript proporciona las estructuras `try` y `catch`. El código que podría generar una excepción se coloca dentro del bloque `try`, y si ocurre una excepción, el control pasa al bloque `catch` donde podemos manejar el error.

```js
try {
  let resultado = 10 / 0;
  console.log("El resultado es " + resultado);
} catch (error) {
  console.error("Ha ocurrido un error: " + error.message);
}
```

En este ejemplo, si bien la operación 10 / 0 no produce una excepción en sí (devuelve `Infinity`), la estructura `try...catch` podría manejar otros tipos de errores que puedan surgir.

```js
try {
  let resultado = variableNoDeclarada; // Error: variableNoDeclarada no está definida
} catch (error) {
  console.error("Ha ocurrido un error: " + error.message);
}
```

Aquí, al intentar usar una variable no declarada, se lanza una excepción, y el bloque `catch` captura el error y muestra un mensaje.

## Uso de finally

La cláusula `finally` se utiliza para definir un bloque de código que se ejecutará independientemente de si ocurre una excepción o no. Esto es útil cuando hay tareas que deben realizarse siempre, como cerrar conexiones o liberar recursos.

```js
try {
  let resultado = 10 / 2;
  console.log("El resultado es " + resultado);
} catch (error) {
  console.error("Ha ocurrido un error: " + error.message);
} finally {
  console.log("Este bloque se ejecuta siempre, con o sin error.");
}
```

En este caso, independientemente de si ocurre un error en el bloque try, el bloque finally siempre se ejecutará.

## Lanzar excepciones con throw

JavaScript permite lanzar excepciones de manera explícita utilizando la palabra clave `throw`. Puedes lanzar cualquier tipo de valor como una excepción, aunque generalmente se lanzan objetos de tipo `Error` o derivados de `Error`, que contienen información sobre el error.

Cuando una excepción es lanzada con `throw`, el flujo del programa se interrumpe y el control pasa inmediatamente al bloque `catch`, si existe.

```js
try {
  let edad = -5;
  if (edad < 0) {
    throw new Error("La edad no puede ser negativa.");
  }
  console.log("Edad válida: " + edad);
} catch (error) {
  console.error("Ha ocurrido un error: " + error.message);
}
```

En este ejemplo, el bloque `try` lanza una excepción si la variable edad tiene un valor negativo. El bloque `catch` captura el error y lo maneja adecuadamente.

## Tipos de errores comunes

JavaScript tiene varios tipos de errores que pueden ser lanzados, muchos de los cuales heredan de la clase base Error. Los más comunes son:

### Error

Es el tipo de error genérico en JavaScript. Puedes lanzar y capturar errores genéricos con `throw` y `catch`.

```js
throw new Error("Este es un error genérico.");
```

### ReferenceError

Este error ocurre cuando intentas acceder a una variable que no está definida.

```js
try {
  console.log(variableNoDeclarada); // ReferenceError: variableNoDeclarada no está definida
} catch (error) {
  console.error(error.name + ": " + error.message);
}
```

### TypeError

Se produce cuando un valor no es del tipo esperado. Por ejemplo, intentar llamar a un valor que no es una función.

```js
try {
  let num = 5;
  num(); // TypeError: num no es una función
} catch (error) {
  console.error(error.name + ": " + error.message);
}
```

### SyntaxError

Este error ocurre cuando el código tiene una sintaxis incorrecta.

```js
try {
  eval("let x = "); // SyntaxError: Se esperaba una expresión después del operador "="
} catch (error) {
  console.error(error.name + ": " + error.message);
}
```

### RangeError

Ocurre cuando un valor está fuera del rango permitido, como pasar un número inválido a una función que espera un rango específico.

```js
try {
  let numero = new Array(-1); // RangeError: el tamaño de la matriz no puede ser negativo
} catch (error) {
  console.error(error.name + ": " + error.message);
}
```

## Propagación de excepciones

Las excepciones en JavaScript se propagan "hacia arriba" en la cadena de llamadas de funciones. Si una función lanza una excepción, y esa excepción no es manejada dentro de la función, se propagará a la siguiente función en la cadena de llamadas hasta que se encuentre un bloque `catch` que la maneje.

```js
function funcionA() {
  funcionB();
}

function funcionB() {
  throw new Error("Ocurrió un error en funcionB.");
}

try {
  funcionA();
} catch (error) {
  console.error("Error capturado: " + error.message);
}
```

Aquí, el error se lanza en `funcionB`, pero es capturado por el bloque `try...catch` en `funcionA`, demostrando cómo las excepciones pueden propagarse a lo largo de múltiples funciones.

## Buenas prácticas en el manejo de excepciones

**Maneja solo los errores que puedas controlar:** No es necesario rodear cada línea de código con un bloque `try...catch`. Usa estos bloques solo cuando sea probable que ocurra un error que puedas manejar correctamente.

**Proporciona mensajes de error claros:** Los mensajes de error deben ser descriptivos y útiles para los desarrolladores que depuren el código.

**Utiliza errores personalizados cuando sea necesario:** Si tienes errores específicos para tu aplicación, considera crear clases de error personalizadas para hacer que el manejo de excepciones sea más claro.
