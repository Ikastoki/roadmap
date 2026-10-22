<h1 align="center">XMLHttpRequest</h1>

<h2>📑 Contenido</h2>

- [XMLHttpRequest](#xmlhttprequest)
- [¿Cómo funciona XMLHttpRequest?](#cómo-funciona-xmlhttprequest)
- [Ejemplo de una Petición GET](#ejemplo-de-una-petición-get)

## XMLHttpRequest

XMLHttpRequest (XHR) es un objeto de JavaScript que te permite realizar peticiones de red a un servidor para obtener datos sin tener que recargar la página completa. Fue la primera tecnología de facto para hacer peticiones asíncronas, lo que sentó las bases para el desarrollo de aplicaciones web dinámicas y modernas.

## ¿Cómo funciona XMLHttpRequest?

1. El proceso de una petición con XHR sigue estos pasos:

1. Crear una instancia de XHR: Creas un nuevo objeto `XMLHttpRequest()`.

1. Abrir la conexión: Usas el método `open()` para configurar la petición. Le especificas el método HTTP (`GET`, `POST`, etc.) y la URL del recurso que quieres solicitar.

1. Configurar un `event listener`: Adjuntas un `listener` al objeto XHR para saber cuándo ha terminado la petición. El evento más común es load.

1. Enviar la petición: Usas el método `send()` para enviar la petición al servidor.

1. Manejar la respuesta: Una vez que la petición se completa, el `event listener` se activa y puedes acceder a los datos de la respuesta.

## Ejemplo de una Petición GET

Este es un ejemplo clásico de cómo usar XHR para obtener datos de una API.

```javascript
// 1. Crear una nueva instancia de XHR
const xhr = new XMLHttpRequest();

// 2. Abrir la conexión (método GET, URL)
xhr.open("GET", "https://jsonplaceholder.typicode.com/posts/1");

// 3. Configurar el listener para el evento `load`
xhr.addEventListener("load", function () {
  // 4. Verificar el estado y el código HTTP
  if (xhr.status === 200) {
    // La petición fue exitosa (código 200)
    console.log("Datos recibidos correctamente:", JSON.parse(xhr.responseText));
  } else {
    // Algo salió mal
    console.error("Error en la petición:", xhr.statusText);
  }
});

// 5. Enviar la petición
xhr.send();
```

> [!NOTE]
> Aunque XMLHttpRequest fue una pieza fundamental para las peticiones asíncronas, la API `Fetch` es ahora el estándar recomendado por su diseño más moderno y su integración con `Promises` y `async/await`.
