<h1 align="center">HTTP</h1>

<h2>📑 Contenido</h2>

- [Módulo http](#módulo-http)
- [Creando un Servidor Básico](#creando-un-servidor-básico)
- [Objetos clave: request y response](#objetos-clave-request-y-response)
  - [Objeto `request` (`req`)](#objeto-request-req)
  - [Objeto `response` (`res`)](#objeto-response-res)

## Módulo http

El módulo http es un módulo nativo de Node.js que te permite crear servidores web y realizar peticiones HTTP sin necesidad de librerías de terceros. Es la base sobre la que se construyen frameworks más avanzados como Express.js.

Con este módulo, puedes:

- Crear un servidor que escuche peticiones en un puerto específico.

- Manejar las peticiones entrantes (`request`).

- Enviar respuestas (`response`) al cliente.

## Creando un Servidor Básico

Aquí tienes un ejemplo de cómo crear un servidor web simple que responde a cualquier petición con un mensaje.

```javascript
const http = require("node:http");

// 1. Definimos el hostname y el puerto donde el servidor escuchará
const hostname = "127.0.0.1"; // localhost
const port = 3000;

// 2. Creamos el servidor. El callback se ejecuta cada vez que llega una petición.
const server = http.createServer((req, res) => {
  // req: objeto Request (contiene información de la petición del cliente)
  // res: objeto Response (se usa para enviar la respuesta al cliente)

  // 3. Configuramos la respuesta del servidor
  res.statusCode = 200; // Código de estado HTTP: 200 OK
  res.setHeader("Content-Type", "text/plain"); // Tipo de contenido de la respuesta

  // 4. Escribimos y enviamos la respuesta
  res.end("Hola Mundo desde Node.js\n");
});

// 5. El servidor se pone a la escucha en el puerto y hostname definidos
server.listen(port, hostname, () => {
  console.log(`El servidor se está ejecutando en http://${hostname}:${port}/`);
});
```

## Objetos clave: request y response

Cada vez que el servidor recibe una petición, el callback recibe dos objetos:

### Objeto `request` (`req`)

Contiene toda la información sobre la petición del cliente.

- `req.method`: El método HTTP de la petición (ej. `'GET'`, `'POST'`).

- `req.url`: La URL a la que se hizo la petición (ej. `'/users'`).

- `req.headers`: Un objeto con las cabeceras de la petición.

- `req.body`: Para peticiones `POST`, contiene el cuerpo de la petición. Es un flujo (`stream`), por lo que debes procesarlo para obtener los datos completos.

### Objeto `response` (`res`)

Se usa para construir y enviar la respuesta al cliente.

- `res.statusCode`: Establece el código de estado HTTP (ej. `200`, `404`, `500`).

- `res.setHeader(nombre, valor)`: Establece una cabecera HTTP.

- `res.write()`: Escribe datos en el cuerpo de la respuesta.

- `res.end()`: Finaliza la respuesta y la envía al cliente.

El módulo http es la base fundamental para entender cómo funciona la comunicación web en Node.js, aunque en la práctica, para proyectos reales, se prefiere usar frameworks como Express.js, que simplifican enormemente esta tarea.
