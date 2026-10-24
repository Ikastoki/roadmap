# URL y URLConnection

Java proporciona soporte nativo para el protocolo HTTP (y otros protocolos URI) desde sus primeras versiones a traves del paquete `java.net`. La clase `URL` permite construir, parsear y abrir conexiones hacia recursos identificados por una URL, mientras que `URLConnection` (y su subclase `HttpURLConnection`) proporcionan el control fino sobre los parametros de la conexion, las cabeceras y el intercambio de datos con servidores web.

## java.net.URL

`URL` representa un Localizador Uniforme de Recursos y proporciona metodos para descomponer una URL en sus componentes constitutivos: protocolo, autoridad, host, puerto, ruta, query y fragmento.

```java
URL url = new URL("https://api.example.com:8443/usuarios?id=123&activo=true#top");

System.out.println("Protocolo: " + url.getProtocol());   // https
System.out.println("Host: "     + url.getHost());        // api.example.com
System.out.println("Puerto: "   + url.getPort());        // 8443
System.out.println("Path: "     + url.getPath());        // /usuarios
System.out.println("Query: "    + url.getQuery());       // id=123&activo=true
System.out.println("Ref: "      + url.getRef());         // top
```

Si el puerto no se especifica en la URL, `getPort()` devuelve -1, y debe obtenerse el puerto por defecto del protocolo con `getDefaultPort()` (80 para HTTP, 443 para HTTPS).

`URL` proporciona dos metodos para obtener el contenido del recurso:

- `openStream()` -- abre una conexion y devuelve un `InputStream` para leer el contenido directamente. Es el metodo mas simple para realizar una peticion GET.
- `openConnection()` -- devuelve un objeto `URLConnection` que permite configurar la conexion antes de establecerla.

```java
// Lectura directa con openStream()
URL url = new URL("https://example.com");
BufferedReader reader = new BufferedReader(
    new InputStreamReader(url.openStream(), StandardCharsets.UTF_8));
reader.lines().forEach(System.out::println);
```

## URLConnection

`URLConnection` es una clase abstracta que gestiona la comunicacion entre la aplicacion y el recurso remoto. Se obtiene a traves de `URL.openConnection()` y proporciona metodos para configurar parametros de la conexion antes de establecerla, y para leer metadatos de la respuesta despues.

**Configuracion previa a la conexion.** Antes de llamar a `connect()`, se pueden configurar diversos parametros:

```java
URL url = new URL("https://api.example.com/data");
URLConnection conn = url.openConnection();

conn.setConnectTimeout(5000);       // Timeout de conexion: 5 segundos
conn.setReadTimeout(10000);          // Timeout de lectura: 10 segundos
conn.setRequestProperty("User-Agent", "Java-Client/1.0");
conn.setRequestProperty("Accept", "application/json");

conn.connect();
```

`setConnectTimeout(int timeout)` establece el tiempo maximo en milisegundos para establecer la conexion. `setReadTimeout(int timeout)` establece el tiempo maximo entre la lectura de dos bloques de datos. Ambos son esenciales para evitar que la aplicacion se cuelgue indefinidamente ante un servidor que no responde.

`setRequestProperty(String key, String value)` anade una cabecera HTTP a la peticion. `getRequestProperty(String key)` recupera el valor de una cabecera previamente establecida. Las cabeceras se envian al servidor en el momento de `connect()`.

**Lectura de la respuesta.** Despues de establecer la conexion, se pueden consultar las cabeceras de respuesta y leer el contenido:

```java
System.out.println("Content-Type: " + conn.getContentType());
System.out.println("Content-Length: " + conn.getContentLengthLong());
System.out.println("Encoding: " + conn.getContentEncoding());

Map<String, List<String>> headers = conn.getHeaderFields();
headers.forEach((key, value) -> System.out.println(key + ": " + value));

InputStream in = conn.getInputStream();
String body = new String(in.readAllBytes(), StandardCharsets.UTF_8);
```

## HttpURLConnection

`HttpURLConnection` es la subclase de `URLConnection` especifica para el protocolo HTTP. Proporciona metodos adicionales para controlar el metodo HTTP, manejar redirecciones, y acceder a codigos de estado y mensajes de error.

```java
URL url = new URL("https://api.example.com/usuarios");
HttpURLConnection httpConn = (HttpURLConnection) url.openConnection();

httpConn.setRequestMethod("GET");
httpConn.setConnectTimeout(5000);
httpConn.setReadTimeout(5000);
httpConn.setInstanceFollowRedirects(true); // Seguir redirecciones automaticamente

int responseCode = httpConn.getResponseCode();

if (responseCode == HttpURLConnection.HTTP_OK) {
    BufferedReader in = new BufferedReader(
        new InputStreamReader(httpConn.getInputStream(), StandardCharsets.UTF_8));
    String response = in.lines().collect(Collectors.joining("\n"));
    in.close();
    System.out.println("Respuesta: " + response);
} else {
    System.out.println("Error: " + responseCode + " " + httpConn.getResponseMessage());
}

httpConn.disconnect();
```

`setRequestMethod(String method)` establece el metodo HTTP: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS. Si no se especifica, el metodo por defecto es GET.

`getResponseCode()` devuelve el codigo de estado HTTP (200, 404, 500, etc.). `getResponseMessage()` devuelve el mensaje asociado ("OK", "Not Found", "Internal Server Error").

`setInstanceFollowRedirects(boolean)` controla si `HttpURLConnection` sigue automaticamente las redirecciones HTTP 3xx. Por defecto es `true`. `getHeaderField("Location")` permite obtener manualmente la URL de redireccion.

> Es importante llamar siempre a `disconnect()` para liberar los recursos de la conexion. Aunque la conexion se cierra automaticamente al leer completamente la respuesta, `disconnect()` asegura que los recursos nativos subyacentes se liberan inmediatamente, especialmente en conexiones persistentes (HTTP keep-alive).

## Ejemplo completo: consulta a API REST

```java
public class ApiClient {
    public static void main(String[] args) throws Exception {
        URL url = new URL("https://jsonplaceholder.typicode.com/posts/1");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();

        conn.setRequestMethod("GET");
        conn.setRequestProperty("Accept", "application/json");
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);

        int status = conn.getResponseCode();
        System.out.println("Status: " + status);

        if (status == 200) {
            String body = new String(conn.getInputStream().readAllBytes(),
                StandardCharsets.UTF_8);
            System.out.println(body);
        } else {
            System.out.println("Error stream: " +
                new String(conn.getErrorStream().readAllBytes(),
                    StandardCharsets.UTF_8));
        }

        conn.disconnect();
    }
}
```
