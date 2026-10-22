<h1 align="center">Fetch API</h1>

<h2>📑 Contenido</h2>

- [Fetch API](#fetch-api)
- [¿Cómo funciona la Fetch API?](#cómo-funciona-la-fetch-api)
- [Peticiones GET simples](#peticiones-get-simples)
- [Peticiones POST y configuración](#peticiones-post-y-configuración)
- [Usando async/await con fetch](#usando-asyncawait-con-fetch)

## Fetch API

La Fetch API es el estándar moderno en JavaScript para realizar peticiones de red. Es la sucesora de XMLHttpRequest y ofrece una forma más potente y flexible de trabajar con recursos de forma asíncrona.

## ¿Cómo funciona la Fetch API?

A diferencia de `XMLHttpRequest` que se basa en eventos, `Fetch` utiliza Promesas. Esto hace que el código sea mucho más limpio, legible y fácil de manejar, especialmente con la sintaxis de async/await.

Una llamada a `fetch()` devuelve una Promesa que se resuelve con un objeto `Response`. Este objeto contiene información sobre la respuesta del servidor, pero no el cuerpo de los datos directamente. Para obtener los datos, debes llamar a un método en el objeto `Response`, como `.json()`, que a su vez devuelve otra Promesa.

## Peticiones GET simples

La forma más básica de usar `fetch` es para una petición `GET`, donde solo necesitas pasar la URL del recurso.

```javascript
fetch("https://api.example.com/data")
  .then((response) => {
    // La primera promesa se resuelve con el objeto Response.
    // Verificamos si la respuesta es exitosa.
    if (!response.ok) {
      throw new Error("La petición no fue exitosa.");
    }
    // Convertimos el cuerpo de la respuesta a JSON. Esto devuelve otra promesa.
    return response.json();
  })
  .then((data) => {
    // La segunda promesa se resuelve con los datos JSON.
    console.log("Datos recibidos:", data);
  })
  .catch((error) => {
    // Capturamos cualquier error en la cadena de promesas (red, servidor, etc.).
    console.error("Hubo un problema con la petición:", error);
  });
```

El objeto `Response` tiene una propiedad `.ok` que es `true` para los códigos de estado HTTP 2xx (éxito). Es una buena práctica verificarla, ya que `fetch` solo lanza un error de red, no un error de HTTP como 404 o 500.

## Peticiones POST y configuración

Para peticiones más complejas como `POST`, necesitas pasar un segundo argumento a `fetch` con un objeto de configuración.

```javascript
const nuevoPost = {
  titulo: "Mi primer post",
  contenido: "Este es el contenido de mi post.",
};

fetch("https://api.example.com/posts", {
  method: "POST", // El método HTTP
  headers: {
    "Content-Type": "application/json", // Informamos al servidor que estamos enviando JSON
  },
  body: JSON.stringify(nuevoPost), // Convertimos el objeto JavaScript a una cadena JSON
})
  .then((response) => response.json())
  .then((data) => {
    console.log("Post creado exitosamente:", data);
  })
  .catch((error) => {
    console.error("Error al crear el post:", error);
  });
```

En este ejemplo, configuramos el método, las cabeceras (headers) y el cuerpo de la petición (body).

## Usando async/await con fetch

Para una sintaxis aún más limpia y legible, puedes combinar `fetch` con `async/await`, que te permite escribir código asíncrono como si fuera síncrono.

```javascript
async function obtenerDatos() {
  try {
    const response = await fetch("https://api.example.com/data");
    if (!response.ok) {
      throw new Error(`Error HTTP: ${response.status}`);
    }
    const data = await response.json();
    console.log("Datos recibidos:", data);
  } catch (error) {
    console.error("Hubo un problema:", error);
  }
}

obtenerDatos();
```

El uso de `async/await` simplifica la cadena de promesas y el manejo de errores.
