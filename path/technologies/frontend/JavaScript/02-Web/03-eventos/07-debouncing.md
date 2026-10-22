<h1 align="center">Debouncing</h1>

<h2>📑 Contenido</h2>

- [¿Qué es el Debouncing?](#qué-es-el-debouncing)
- [¿Cómo funciona?](#cómo-funciona)
- [Ejemplo Práctico](#ejemplo-práctico)

## ¿Qué es el Debouncing?

El debouncing es una técnica que se utiliza para retrasar la ejecución de una función hasta que un evento deja de ser disparado durante un periodo de tiempo determinado. Es especialmente útil para eventos que se disparan muy rápidamente y de forma repetida, como:

- Escribir en un campo de texto (`keyup`, `input`).

- Cambiar el tamaño de la ventana (`resize`).

- Desplazarse por una página (`scroll`).

- Arrastrar un elemento (`drag`).

Sin el debouncing, cada vez que el evento se dispara (por ejemplo, cada letra que escribes), se ejecutaría una función costosa, lo que podría sobrecargar el navegador y hacer la interfaz de usuario lenta o no responsiva. Con el debouncing, la función solo se ejecuta una vez que el usuario ha terminado su interacción.

## ¿Cómo funciona?

El debouncing se implementa con una función de alto orden que envuelve a la función que queremos ejecutar. Esta función utiliza `setTimeout()` para programar la ejecución de la función original.

1. Cuando el evento se dispara por primera vez, el debounce inicia un temporizador.

2. Si el evento se vuelve a disparar antes de que el temporizador termine, el temporizador anterior se cancela (`clearTimeout()`) y se inicia uno nuevo.

3. Solo cuando el temporizador finaliza sin ser interrumpido, la función original se ejecuta.

Esto asegura que la función solo se ejecute una vez al final de una ráfaga de eventos.

## Ejemplo Práctico

Imagina que tienes un campo de búsqueda y quieres hacer una llamada a una API cada vez que el usuario escribe. Si lo haces sin debouncing, cada pulsación de tecla hará una llamada, lo cual es ineficiente.

Sin Debouncing (ineficiente):

```javascript
const inputElement = document.getElementById("miInput");

inputElement.addEventListener("input", function () {
  console.log("Haciendo llamada a la API..."); // Se ejecuta en cada pulsación de tecla
});
```

Con Debouncing (eficiente):

```javascript
// Función genérica de debounce
function debounce(fn, delay) {
  let timeoutID;
  return function (...args) {
    clearTimeout(timeoutID); // Cancela el temporizador anterior
    timeoutID = setTimeout(() => {
      fn.apply(this, args); // Ejecuta la función original
    }, delay);
  };
}

const inputElement = document.getElementById("miInput");

// La función a ejecutar, que es la que queremos retrasar
function miFuncionDeBusqueda() {
  console.log("Realizando búsqueda en la API...");
}

// Envolvemos la función con debounce para crear una versión optimizada
const funcionOptimizada = debounce(miFuncionDeBusqueda, 500);

inputElement.addEventListener("input", funcionOptimizada);
```

En este ejemplo, la función de búsqueda solo se ejecutará 500 milisegundos después de que el usuario haya dejado de escribir, lo que reduce drásticamente el número de llamadas a la API.
