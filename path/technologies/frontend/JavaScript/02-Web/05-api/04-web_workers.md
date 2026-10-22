<h1 align="center">Web workers</h1>

<h2>📑 Contenido</h2>

- [Web workers](#web-workers)
- [¿Qué son y por qué son importantes?](#qué-son-y-por-qué-son-importantes)
- [¿Cómo funcionan?](#cómo-funcionan)
- [Ejemplo Práctico](#ejemplo-práctico)
- [Limitaciones de los Web Workers](#limitaciones-de-los-web-workers)

## Web workers

Los Web Workers son una parte fundamental de la API de JavaScript en el navegador que te permiten ejecutar scripts en un hilo de ejecución secundario, separado del hilo principal de la página.

## ¿Qué son y por qué son importantes?

El navegador, por defecto, ejecuta todo tu código JavaScript en un único hilo, llamado el hilo principal. Este hilo es responsable de renderizar la interfaz de usuario, manejar los eventos (clics, scrolls, etc.) y ejecutar tus scripts. Si ejecutas una tarea muy pesada y de larga duración en este hilo, la interfaz de usuario se bloqueará, lo que puede hacer que la página se congele, no responda o que el navegador muestre una alerta de "la página no responde".

Un Web Worker soluciona este problema. Te permite realizar tareas intensivas en la CPU (como cálculos complejos, procesamiento de datos grandes o peticiones de red largas) en un hilo secundario sin afectar la fluidez del hilo principal.

## ¿Cómo funcionan?

- Creación: Creas una instancia de `Worker` y le pasas la URL de un archivo JavaScript que contendrá el código del worker.

```javascript
const miWorker = new Worker("worker.js");
```

- Comunicación: El hilo principal y el worker se comunican a través de un sistema de mensajes.

  - Hilo principal al worker: Usas el método `postMessage(`) para enviar datos al worker.

  - Worker al hilo principal: El worker también usa `postMessage()` para enviar mensajes de vuelta.

- Manejo de mensajes: Tanto el hilo principal como el worker deben usar el método `onmessage` o `addEventListener('message', ...)` para escuchar los mensajes que se envían entre sí.

## Ejemplo Práctico

Imagina que tienes que hacer un cálculo muy largo en tu página.

worker.js (el script del worker):

```javascript
self.onmessage = function (e) {
  const numero = e.data;
  let resultado = 0;
  for (let i = 0; i < numero; i++) {
    resultado += i;
  }
  // Envía el resultado de vuelta al hilo principal
  self.postMessage(resultado);
};
```

index.html (el script en el hilo principal):

```html
<button id="calcular">Calcular</button>
<p>Resultado: <span id="resultado"></span></p>
```

```javascript
const boton = document.getElementById("calcular");
const spanResultado = document.getElementById("resultado");

const miWorker = new Worker("worker.js");

boton.addEventListener("click", function () {
  console.log("Iniciando cálculo en el worker...");
  miWorker.postMessage(10000000000); // Envía un número al worker
});

// Escucha la respuesta del worker
miWorker.onmessage = function (e) {
  spanResultado.textContent = e.data;
  console.log("Cálculo finalizado.");
};
```

En este ejemplo, cuando haces clic en el botón, el hilo principal no se bloquea. El cálculo se realiza en el worker en segundo plano, y la página sigue siendo responsiva. Una vez que el worker termina, envía el resultado de vuelta al hilo principal para que se muestre.

## Limitaciones de los Web Workers

- No pueden acceder al DOM: Los workers no tienen acceso a la ventana (`window`), al documento (`document`) ni a las variables del hilo principal, lo que garantiza su aislamiento.

- Comunicación asíncrona: La comunicación entre el worker y el hilo principal siempre se realiza a través de mensajes asíncronos, lo que obliga a diseñar tu aplicación en torno a este modelo.
