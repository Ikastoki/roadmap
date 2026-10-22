<h1 align='center'>Ajax</h1>

<h2>📑 Contenido</h2>

- [Ajax](#ajax)
- [¿Qué es AJAX?](#qué-es-ajax)
  - [Componentes principales de AJAX](#componentes-principales-de-ajax)
- [¿Cómo funciona AJAX?](#cómo-funciona-ajax)
- [Implementación de AJAX con XMLHttpRequest](#implementación-de-ajax-con-xmlhttprequest)
- [Implementación de AJAX con Fetch API](#implementación-de-ajax-con-fetch-api)
- [AJAX con Async/Await](#ajax-con-asyncawait)
- [Métodos HTTP en AJAX](#métodos-http-en-ajax)

## Ajax

AJAX (Asynchronous JavaScript and XML) es una técnica utilizada para realizar solicitudes al servidor sin recargar la página web. Aunque en sus orígenes usaba XML como formato de datos, hoy en día suele emplear otros formatos, como JSON o HTML. El objetivo principal de AJAX es mejorar la experiencia del usuario en las aplicaciones web, permitiendo que se actualicen dinámicamente partes del contenido sin necesidad de refrescar toda la página.

Este capítulo cubrirá los conceptos clave de AJAX, desde cómo funciona hasta cómo se implementa utilizando varias herramientas en JavaScript.

## ¿Qué es AJAX?

El término AJAX se refiere a un conjunto de tecnologías que permiten la actualización asíncrona de datos entre el cliente (navegador) y el servidor, lo que mejora la velocidad y la fluidez de las páginas web modernas. AJAX utiliza la capacidad de JavaScript para hacer solicitudes HTTP al servidor sin tener que recargar la página.

### Componentes principales de AJAX

- **JavaScript:** El lenguaje de programación utilizado para enviar solicitudes asíncronas al servidor.
- **XMLHttpRequest:** El objeto nativo de JavaScript que permite realizar solicitudes HTTP de manera asíncrona.
- **JSON o XML:** Formatos comunes para intercambiar datos entre el cliente y el servidor (aunque otros formatos como HTML o texto plano también son posibles).
- **HTML/CSS:** Se actualizan dinámicamente en función de la respuesta recibida del servidor.
- **Servidor Web:** Procesa las solicitudes y envía respuestas basadas en la lógica del lado del servidor.

## ¿Cómo funciona AJAX?

El flujo de trabajo básico de AJAX se puede descomponer en los siguientes pasos:

1. **Evento del Usuario:** El usuario interactúa con la página (por ejemplo, haciendo clic en un botón).
1. **Solicitud Asíncrona:** JavaScript crea una solicitud HTTP usando el objeto XMLHttpRequest o las modernas APIs fetch.
1. **Envío al Servidor:** La solicitud se envía al servidor sin interrumpir la página actual.
1. **Procesamiento del Servidor:** El servidor recibe la solicitud, la procesa y genera una respuesta (normalmente en formato JSON o XML).
1. **Respuesta Asíncrona:** El servidor devuelve la respuesta a la página sin causar un refresco completo.
1. **Actualización del DOM:** JavaScript recibe la respuesta y actualiza el contenido de la página de forma dinámica.

Este proceso mejora la experiencia del usuario al evitar la recarga completa de la página, permitiendo que solo se actualicen las partes necesarias.

## Implementación de AJAX con XMLHttpRequest

Uno de los primeros enfoques para implementar AJAX fue utilizando el objeto `XMLHttpRequest`. Aunque este enfoque sigue siendo válido, ha sido superado en simplicidad y legibilidad por nuevas tecnologías como Fetch API y Axios. Sin embargo, conocer `XMLHttpRequest` es importante para entender cómo funciona AJAX a bajo nivel.

```js
const xhr = new XMLHttpRequest();

// Inicializar una solicitud
xhr.open("GET", "https://api.ejemplo.com/datos", true);

// Manejar la respuesta del servidor
xhr.onload = function () {
  if (xhr.status === 200) {
    console.log(xhr.responseText); // Procesa la respuesta
  } else {
    console.error("Error en la solicitud", xhr.status);
  }
};

// Enviar la solicitud
xhr.send();
```

- **open():** Inicia la solicitud. El primer argumento es el método HTTP (como GET o POST), el segundo es la URL, y el tercero indica si la solicitud es asíncrona (true).
- **onload():** Se ejecuta cuando la solicitud ha sido completada. Aquí es donde se maneja la respuesta.
- **status:** Código HTTP de respuesta (200 significa éxito).
- **responseText:** El cuerpo de la respuesta, normalmente en formato JSON o texto.

## Implementación de AJAX con Fetch API

Fetch API es una forma moderna y más amigable de realizar solicitudes HTTP asíncronas. A diferencia de `XMLHttpRequest`, `fetch` es más flexible y retorna promesas, lo que facilita su integración con las funciones `async` y `await`.

> [!NOTE]
> Si quieres saber más de `Fetch API` puedes ir a la sección de APIs en "El Navegador".

```js
fetch("https://api.ejemplo.com/datos")
  .then((respuesta) => {
    if (!respuesta.ok) {
      throw new Error("Error en la solicitud");
    }
    return respuesta.json();
  })
  .then((datos) => {
    console.log(datos); // Procesar los datos recibidos
  })
  .catch((error) => {
    console.error("Hubo un error:", error);
  });
```

- **fetch():** Inicia la solicitud. Retorna una promesa que se resuelve cuando el servidor responde.
- **respuesta.ok:** Verifica si la solicitud fue exitosa (código de estado 200-299).
- **respuesta.json():** Convierte la respuesta a formato JSON.
- **.then():** Procesa la respuesta cuando la promesa se resuelve.
- **.catch():** Captura cualquier error durante el proceso de la solicitud.

## AJAX con Async/Await

Utilizando `async` y `await`, el código asíncrono con `fetch` puede hacerse más claro y legible.

```js
async function obtenerDatos() {
  try {
    const respuesta = await fetch("https://api.ejemplo.com/datos");
    if (!respuesta.ok) {
      throw new Error("Error en la solicitud");
    }
    const datos = await respuesta.json();
    console.log(datos); // Procesar los datos
  } catch (error) {
    console.error("Hubo un error:", error);
  }
}

obtenerDatos();
```

- **async:** Marca la función como asíncrona, lo que permite usar await dentro de ella.
- **await:** Pausa la ejecución de la función hasta que la promesa se resuelva.
- **try/catch:** Se utiliza para manejar errores de manera más intuitiva.

## Métodos HTTP en AJAX

AJAX permite realizar todo tipo de solicitudes HTTP, no solo GET. Algunos métodos comunes son:

- **GET:** Para obtener datos del servidor.
- **POST:** Para enviar datos al servidor.
- **PUT:** Para actualizar recursos en el servidor.
- **DELETE:** Para eliminar recursos del servidor.

Ejemplo con POST utilizando fetch:

```js
async function enviarDatos(datos) {
  try {
    const respuesta = await fetch("https://api.ejemplo.com/enviar", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(datos),
    });

    const resultado = await respuesta.json();
    console.log("Datos enviados:", resultado);
  } catch (error) {
    console.error("Error al enviar datos:", error);
  }
}

enviarDatos({ nombre: "Juan", edad: 30 });
```

En este ejemplo, `fetch` envía una solicitud POST con un objeto JSON en el cuerpo de la solicitud.
