# Conceptos de APIs REST

REST (Representational State Transfer) es un estilo arquitectonico para disenar sistemas de red que define un conjunto de restricciones con las que crear servicios web escalables y desacoplados. A diferencia de SOAP, REST no impone un formato de mensaje fijo ni requiere un esquema WSDL; se apoya en los principios fundamentales del protocolo HTTP y en una separacion clara entre cliente y servidor. La adopcion masiva de REST en la industria responde a su simplicidad, su capacidad de cacheo y su facil integracion con cualquier lenguaje que hable HTTP.

## Recursos y URIs

En REST, todo es un recurso: un usuario, una factura, un archivo, un estado. Cada recurso se identifica de forma unica mediante una URI (Uniform Resource Identifier). La URI debe ser sustantiva y representar una entidad, no una accion. Por ejemplo, `/api/usuarios/42` identifica al usuario con id 42; `/api/usuarios` identifica la coleccion de usuarios. Una URI como `/api/obtenerUsuario` rompe el modelo REST porque describe una accion en lugar de un recurso.

La estructura jerarquica de las URIs permite relaciones: `/api/usuarios/42/pedidos` representa los pedidos del usuario 42. Los parametros de consulta (`?page=2&limit=10`) se usan para filtrado, paginacion u ordenacion, no para identificar recursos.

## Metodos HTTP

Cada recurso se manipula a traves de metodos HTTP estandar. El protocolo define el significado semantico de cada operacion:

- **GET**: recupera una representacion del recurso. Es seguro (no modifica el estado) e idempotente (multiples peticiones identicas producen el mismo resultado).
- **POST**: crea un nuevo recurso. No es idempotente; cada peticion crea un recurso distinto.
- **PUT**: reemplaza completamente un recurso existente o lo crea si no existe. Es idempotente.
- **PATCH**: aplica una modificacion parcial a un recurso. No es necesariamente idempotente (depende de la implementacion).
- **DELETE**: elimina un recurso. Es idempotente.

Un diseno RESTful correcto utiliza el metodo adecuado para cada operacion. Usar POST para todo (o solo GET y POST) desvirtua el modelo y elimina las garantias de cacheabilidad e idempotencia que ofrece HTTP.

```java
// Ejemplo conceptual: no es codigo ejecutable
GET    /api/usuarios       -> listar usuarios
GET    /api/usuarios/42    -> obtener usuario 42
POST   /api/usuarios       -> crear usuario
PUT    /api/usuarios/42    -> reemplazar usuario 42
PATCH  /api/usuarios/42    -> modificar parcialmente usuario 42
DELETE /api/usuarios/42    -> eliminar usuario 42
```

## Codigos de estado HTTP

La respuesta del servidor incluye un codigo de estado de tres digitos que informa del resultado de la operacion. Las categorias principales son:

- **1xx (Informativos)**: raramente usados en APIs REST. 101 Switching Protocols se emplea en WebSockets.
- **2xx (Exito)**: 200 OK (exito generico), 201 Created (recurso creado via POST), 204 No Content (exito sin cuerpo, tipico en DELETE).
- **3xx (Redireccion)**: 301 Moved Permanently, 304 Not Modified (cacheo condicional con ETag).
- **4xx (Error del cliente)**: 400 Bad Request (sintaxis invalida), 401 Unauthorized (requiere autenticacion), 403 Forbidden (sin permisos), 404 Not Found, 405 Method Not Allowed, 409 Conflict (violacion de estado), 422 Unprocessable Entity (validacion de negocio).
- **5xx (Error del servidor)**: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable.

> Un error comun en APIs REST es devolver siempre 200 OK con un campo `error` en el cuerpo. Esto impide que intermediarios como proxies o balanceadores interpreten correctamente el resultado. Los codigos HTTP deben reflejar el estado real de la operacion.

## Stateless

REST exige que el servidor no mantenga estado de sesion entre peticiones. Cada peticion del cliente contiene toda la informacion necesaria para ser procesada (autenticacion, contexto, parametros). Esto permite escalar horizontalmente cualquier nodo del servidor sin necesidad de compartir estado de sesion.

> El stateless no implica que la aplicacion no tenga estado; implica que el servidor no almacena estado de dialogo entre peticiones. El estado del recurso (los datos) se persiste en una base de datos u otro almacenamiento compartido.

## Representaciones (JSON/XML)

El cliente y el servidor negocian el formato de intercambio mediante el encabezado `Content-Type` en la peticion y `Accept` en la respuesta. Los formatos mas comunes son JSON (ligero, nativo en JavaScript, legible) y XML (verboso pero con soporte de esquemas XSD). REST no exige un formato concreto; la representacion es un detalle de implementacion intercambiable.

```http
GET /api/usuarios/42 HTTP/1.1
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 42,
  "nombre": "Ana Garcia",
  "email": "ana@example.com"
}
```
