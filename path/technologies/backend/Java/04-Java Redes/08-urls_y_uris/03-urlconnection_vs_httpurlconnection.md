# URLConnection vs HttpURLConnection

`URLConnection` es la clase abstracta base para todas las conexiones que Java establece a traves de una URL, mientras que `HttpURLConnection` extiende sus funcionalidades para el protocolo HTTP. Esta jerarquia permite escribir codigo generico independiente del protocolo y, al mismo tiempo, acceder a capacidades especificas cuando se conoce el esquema subyacente.

## URLConnection: la clase base

Toda conexion comienza con `URL.openConnection()`, que devuelve una instancia de `URLConnection` (o una subclase concreta segun el protocolo). El objeto no establece la conexion real hasta que se invoca `connect()` o alguna operacion que implicitamente la requiera.

```java
URL url = new URL("https://api.example.com/data");
URLConnection conn = url.openConnection();
conn.setRequestProperty("Accept", "application/json");
conn.setRequestProperty("User-Agent", "JavaClient/1.0");
conn.connect(); // opcional, getInputStream() lo hace automaticamente
```

### getInputStream

El metodo `getInputStream()` devuelve un flujo para leer la respuesta del servidor. Antes de la primera lectura, establece la conexion si aun no se ha hecho. Para errores HTTP (4xx/5xx), `getInputStream()` lanza `IOException`; en esos casos debe usarse `getErrorStream()` en `HttpURLConnection`.

```java
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(conn.getInputStream()))) {
    br.lines().forEach(System.out::println);
}
```

### setRequestProperty

Permite configurar cabeceras HTTP en la peticion. Las propiedades deben establecerse antes de `connect()` o de cualquier operacion que desencadene la conexion. `getRequestProperty()` y `getHeaderFields()` permiten inspeccionar cabeceras de la peticion y la respuesta respectivamente.

```java
conn.setRequestProperty("Content-Type", "application/json");
conn.setRequestProperty("Authorization", "Bearer " + token);
```

## HttpURLConnection: funcionalidad especifica HTTP

Cuando el protocolo de la URL es HTTP o HTTPS, `openConnection()` devuelve una instancia de `HttpURLConnection`. Se obtiene casteando la referencia de la clase base.

```java
URL url = new URL("https://api.example.com/users");
HttpURLConnection httpConn = (HttpURLConnection) url.openConnection();
```

### setRequestMethod

Define el verbo HTTP de la peticion: GET, POST, PUT, DELETE, HEAD, OPTIONS, TRACE o PATCH (este ultimo disponible desde Java 11+ con `HttpClient` nativo). Por defecto es GET.

```java
httpConn.setRequestMethod("POST");
httpConn.setDoOutput(true); // necesario para enviar cuerpo
try (OutputStream os = httpConn.getOutputStream()) {
    os.write("{\"name\":\"Alice\"}".getBytes(StandardCharsets.UTF_8));
}
```

Al usar metodos que envian cuerpo (POST, PUT, PATCH), debe activarse `setDoOutput(true)`. El output stream se obtiene despues de configurar la peticion pero antes de leer la respuesta.

### getResponseCode y getResponseMessage

`getResponseCode()` devuelve el codigo de estado HTTP (200, 404, 500, etc.) como entero, mientras que `getResponseMessage()` entrega el mensaje asociado ("OK", "Not Found", "Internal Server Error" respectivamente).

```java
int code = httpConn.getResponseCode();
String message = httpConn.getResponseMessage();
if (code == HttpURLConnection.HTTP_OK) {
    // leer cuerpo normal
} else {
    // leer error stream
    try (InputStream err = httpConn.getErrorStream()) {
        // procesar error
    }
}
```

### Seguimiento de redirecciones

`HttpURLConnection` sigue automaticamente las redirecciones HTTP (301, 302, 303, 307, 308) hasta un maximo configurable. `setInstanceFollowRedirects(false)` permite deshabilitar este comportamiento para manejarlas manualmente.

```java
httpConn.setInstanceFollowRedirects(false);
int code = httpConn.getResponseCode();
if (code == HttpURLConnection.HTTP_MOVED_TEMP) {
    String newLocation = httpConn.getHeaderField("Location");
    System.out.println("Redirigir a: " + newLocation);
}
```

La configuracion global `HttpURLConnection.setFollowRedirects(boolean)` aplica a todas las conexiones HTTP de la JVM, mientras que `setInstanceFollowRedirects` solo afecta a la instancia actual.

### disconnect

Aunque `URLConnection` no tiene un metodo `disconnect()`, `HttpURLConnection` lo implementa para liberar recursos del socket subyacente. Es especialmente util en aplicaciones de larga duracion o con muchas conexiones.

```java
httpConn.disconnect();
```

Internamente, `disconnect()` marca la conexion como cerrada y permite que el socket se reutilice o libere segun la politica del gestor de conexiones del sistema.

### Metodos adicionales utiles

```java
// Tiempos de espera
conn.setConnectTimeout(5000);     // 5 segundos para establecer conexion
conn.setReadTimeout(10000);       // 10 segundos para leer datos

// Cache
conn.setUseCaches(false);
conn.setDefaultUseCaches(false);

// Tamanio de contenido
int contentLength = conn.getContentLengthLong();
String contentType = conn.getContentType();
```

> A partir de Java 11, `java.net.http.HttpClient` ofrece una API mas moderna, fluida y con soporte nativo para HTTP/2 y WebSockets, relegando `HttpURLConnection` a proyectos legacy. Sin embargo, `HttpURLConnection` sigue siendo la opcion en entornos Android y codigo base existente.
