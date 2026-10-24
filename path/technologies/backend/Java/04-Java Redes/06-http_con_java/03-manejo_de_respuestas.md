# Manejo de respuestas

El manejo de respuestas HTTP en Java con `HttpURLConnection` abarca tres aspectos fundamentales: la interpretacion de los codigos de estado, la lectura de las cabeceras de respuesta y la gestion de situaciones de error. Un manejo incorrecto de estos elementos es la causa mas frecuente de bugs en clientes HTTP basados en la API estandar de Java.

## Codigos de estado HTTP

HTTP define cinco clases de codigos de respuesta, identificadas por el primer digito. Las mas relevantes para el desarrollo de API REST son:

**2xx Exito.**
- `200 OK` -- la peticion se proceso correctamente. Es la respuesta estandar para GET y POST exitosos.
- `201 Created` -- la peticion POST creo un nuevo recurso. Tipicamente incluye la cabecera `Location` con la URL del recurso creado.
- `204 No Content` -- la peticion se proceso correctamente pero no hay contenido que devolver. Comun en respuestas DELETE y PUT exitosas.

**3xx Redireccion.**
- `301 Moved Permanently` -- el recurso se ha movido permanentemente a una nueva URL indicada en `Location`.
- `302 Found` -- el recurso se encuentra temporalmente en otra URL.
- `304 Not Modified` -- el recurso no ha cambiado desde la ultima peticion (util con cabeceras condicionales como `If-Modified-Since`).

**4xx Error del cliente.**
- `400 Bad Request` -- la peticion esta mal formada. Tipicamente JSON invalido o parametros incorrectos.
- `401 Unauthorized` -- autenticacion requerida. El cliente debe proporcionar credenciales validas.
- `403 Forbidden` -- el cliente esta autenticado pero no tiene permisos para acceder al recurso.
- `404 Not Found` -- el recurso solicitado no existe.
- `405 Method Not Allowed` -- el metodo HTTP utilizado no esta soportado para esa URL.
- `409 Conflict` -- conflicto con el estado actual del recurso (ej: version desactualizada).
- `422 Unprocessable Entity` -- la peticion tiene una sintaxis correcta pero errores semanticos (ej: validacion de negocio fallida).
- `429 Too Many Requests` -- el cliente ha superado el limite de peticiones (rate limiting).

**5xx Error del servidor.**
- `500 Internal Server Error` -- error generico del servidor.
- `502 Bad Gateway` -- el servidor actuo como proxy y recibio una respuesta invalida del servidor upstream.
- `503 Service Unavailable` -- el servidor no puede manejar la peticion temporalmente (mantenimiento, sobrecarga).
- `504 Gateway Timeout` -- el servidor upstream no respondio a tiempo.

```java
int code = httpConn.getResponseCode();

switch (code) {
    case HttpURLConnection.HTTP_OK:
    case HttpURLConnection.HTTP_CREATED:
        // Procesar respuesta exitosa
        break;
    case HttpURLConnection.HTTP_NO_CONTENT:
        // No hay contenido que procesar
        break;
    case HttpURLConnection.HTTP_UNAUTHORIZED:
        // Renovar token o solicitar credenciales
        break;
    case HttpURLConnection.HTTP_NOT_FOUND:
        // Recurso no existe
        break;
    case 429:
        // Rate limit superado, esperar antes de reintentar
        break;
    default:
        // Error inesperado
}
```

## Lectura de cabeceras de respuesta

Despues de establecer la conexion y antes de leer el cuerpo, se pueden consultar las cabeceras que el servidor envio en la respuesta. `getHeaderField(String name)` devuelve el valor de una cabecera especifica. `getHeaderFields()` devuelve un `Map<String, List<String>>` con todas las cabeceras.

```java
// Cabeceras individuales
String contentType = httpConn.getHeaderField("Content-Type");
long contentLength = httpConn.getHeaderFieldLong("Content-Length", -1);
String etag         = httpConn.getHeaderField("ETag");
String lastModified = httpConn.getHeaderField("Last-Modified");
String location     = httpConn.getHeaderField("Location");

// Todas las cabeceras
Map<String, List<String>> allHeaders = httpConn.getHeaderFields();
for (Map.Entry<String, List<String>> entry : allHeaders.entrySet()) {
    System.out.println(entry.getKey() + ": " + String.join(", ", entry.getValue()));
}
```

Cabeceras de respuesta tipicas que interesan en un cliente HTTP:

- `Content-Type` -- tipo MIME del cuerpo de respuesta (`application/json`, `text/html`, `application/octet-stream`).
- `Content-Length` -- tamano del cuerpo en bytes. Util para validar que se recibieron todos los datos.
- `ETag` -- identificador de version del recurso. Util para cache condicional con `If-None-Match`.
- `Last-Modified` -- fecha de ultima modificacion del recurso.
- `Set-Cookie` -- cookies que el servidor solicita almacenar.
- `Location` -- URL de redireccion (3xx) o URL del recurso creado (201).
- `Retry-After` -- tiempo que el cliente debe esperar antes de reintentar (usado con 429 y 503).

> `getHeaderFieldLong("Content-Length", -1)` es preferible a `getContentLength()` porque este ultimo devuelve `int`, que se desborda para contenidos mayores de 2 GB.

## Manejo de errores: getErrorStream vs getInputStream

Cuando el codigo de respuesta HTTP indica un error (4xx o 5xx), `getInputStream()` lanza una `IOException`. Sin embargo, el servidor puede incluir un cuerpo en la respuesta de error con detalles adicionales (un JSON con mensaje de error). Para leer este cuerpo debe utilizarse `getErrorStream()`.

```java
int statusCode = httpConn.getResponseCode();
String responseBody;

if (statusCode >= 200 && statusCode < 300) {
    // Respuesta exitosa
    try (InputStream in = httpConn.getInputStream()) {
        responseBody = new String(in.readAllBytes(), StandardCharsets.UTF_8);
    }
} else {
    // Respuesta de error
    try (InputStream err = httpConn.getErrorStream()) {
        if (err != null) {
            responseBody = new String(err.readAllBytes(), StandardCharsets.UTF_8);
        } else {
            responseBody = "Sin cuerpo de error";
        }
    }
}
```

`getErrorStream()` puede devolver `null` si el servidor no envio cuerpo en la respuesta de error. Es importante verificar que no sea `null` antes de leerlo.

## Patron de manejo completo de respuestas

La siguiente clase encapsula el manejo completo de una respuesta HTTP, incluyendo lectura correcta tanto para respuestas exitosas como para errores:

```java
public record HttpResponse(int statusCode, Map<String, List<String>> headers,
                           String body) {

    public boolean isSuccess() {
        return statusCode >= 200 && statusCode < 300;
    }

    public static HttpResponse from(HttpURLConnection conn) throws IOException {
        int statusCode = conn.getResponseCode();
        Map<String, List<String>> headers = conn.getHeaderFields();
        String body;

        try (InputStream in = (statusCode >= 200 && statusCode < 300)
                ? conn.getInputStream()
                : conn.getErrorStream()) {
            if (in != null) {
                body = new String(in.readAllBytes(), StandardCharsets.UTF_8);
            } else {
                body = "";
            }
        }

        return new HttpResponse(statusCode, headers, body);
    }
}
```

## Consideraciones adicionales

La lectura del cuerpo de respuesta debe hacerse antes de llamar a `disconnect()`. Una vez que se desconecta, los flujos quedan invalidados y cualquier intento de lectura lanzara una excepcion.

En respuestas grandes, `readAllBytes()` carga todo el contenido en memoria, lo que puede ser problematico. Para respuestas que pueden ser grandes (decenas de megabytes o mas), es preferible leer el `InputStream` en bloques o utilizar un buffer temporal en disco.

Para conexiones HTTPS, `HttpURLConnection` delega automaticamente en el proveedor JSSE (Java Secure Socket Extension). Cuando se trabaja con certificados autofirmados o de autoridades no estandar, es necesario configurar un `SSLContext` personalizado o un `HostnameVerifier` que acepte la conexion.

> A partir de Java 11, la API `HttpClient` (java.net.http) reemplaza a `HttpURLConnection` con un diseno mas moderno, que soporta HTTP/2, WebSockets, peticiones reactivas con `CompletableFuture` y un manejo mas limpio de respuestas y errores. `HttpURLConnection` sigue siendo util para entornos donde no se puede actualizar la version de Java o para codigos legacy que no pueden migrar.
