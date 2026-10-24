# Peticiones HTTP

`HttpURLConnection` soporta todos los metodos HTTP estandar: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS y TRACE. Cada metodo implica una configuracion diferente de la conexion, especialmente en lo relativo al envio de cuerpo en la peticion y al manejo de la respuesta. A continuacion se presentan ejemplos completos de los metodos mas utilizados en API REST.

## GET

GET es el metodo mas simple. No requiere envio de cuerpo en la peticion, solo cabeceras. Se configura llamando a `setRequestMethod("GET")` y opcionalmente anadiendo cabeceras como `Accept` o `Authorization`.

```java
public static String doGet(String urlStr) throws IOException {
    URL url = new URL(urlStr);
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();

    conn.setRequestMethod("GET");
    conn.setRequestProperty("Accept", "application/json");
    conn.setRequestProperty("Authorization", "Bearer token123");
    conn.setConnectTimeout(5000);
    conn.setReadTimeout(5000);

    int status = conn.getResponseCode();
    String body;

    if (status >= 200 && status < 300) {
        body = new String(conn.getInputStream().readAllBytes(),
            StandardCharsets.UTF_8);
    } else {
        body = new String(conn.getErrorStream().readAllBytes(),
            StandardCharsets.UTF_8);
    }

    conn.disconnect();
    return body;
}
```

Para GET con parametros en la query, estos deben codificarse e incluirse en la propia URL:

```java
String nombre = URLEncoder.encode("Juan Perez", StandardCharsets.UTF_8);
String email  = URLEncoder.encode("juan@example.com", StandardCharsets.UTF_8);
String urlStr = "https://api.example.com/usuarios?nombre=" + nombre + "&email=" + email;
```

`URLEncoder.encode()` es indispensable para evitar caracteres no permitidos en la URL, como espacios, simbolos o caracteres Unicode.

## POST (JSON body)

POST requiere enviar datos en el cuerpo de la peticion, tipicamente en formato JSON o formulario URL-encoded. Para ello debe activarse `setDoOutput(true)`, que permite escribir en el flujo de salida de la conexion.

```java
public static String doPost(String urlStr, String jsonBody) throws IOException {
    URL url = new URL(urlStr);
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();

    conn.setRequestMethod("POST");
    conn.setRequestProperty("Content-Type", "application/json; charset=UTF-8");
    conn.setRequestProperty("Accept", "application/json");
    conn.setDoOutput(true);
    conn.setConnectTimeout(5000);
    conn.setReadTimeout(5000);

    // Enviar cuerpo JSON
    try (OutputStream os = conn.getOutputStream()) {
        byte[] input = jsonBody.getBytes(StandardCharsets.UTF_8);
        os.write(input, 0, input.length);
    }

    int status = conn.getResponseCode();
    String responseBody;

    if (status >= 200 && status < 300) {
        responseBody = new String(conn.getInputStream().readAllBytes(),
            StandardCharsets.UTF_8);
    } else {
        responseBody = new String(conn.getErrorStream().readAllBytes(),
            StandardCharsets.UTF_8);
    }

    conn.disconnect();
    return responseBody;
}
```

Se puede invocar con un JSON como cuerpo:

```java
String json = """
    {
        "nombre": "Juan Perez",
        "email": "juan@example.com",
        "edad": 30
    }
    """;
String respuesta = doPost("https://api.example.com/usuarios", json);
```

> `setDoOutput(true)` cambia implicitamente el metodo a POST si no se ha llamado a `setRequestMethod()`. Aunque es buena practica especificar el metodo explicitamente, es importante saber que `setDoOutput(true)` fuerza internamente el uso de POST.

Para formularios URL-encoded en lugar de JSON, el Content-Type debe ser `application/x-www-form-urlencoded` y el cuerpo debe codificarse:

```java
conn.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");

String formData = "nombre=" + URLEncoder.encode("Juan Perez", UTF_8)
    + "&email=" + URLEncoder.encode("juan@example.com", UTF_8);

try (OutputStream os = conn.getOutputStream()) {
    os.write(formData.getBytes(StandardCharsets.UTF_8));
}
```

## PUT

PUT funciona de forma similar a POST pero con semantica de reemplazo. Se utiliza para actualizar un recurso existente o crearlo si no existe en una URL especifica. La configuracion es identica a POST, salvo el metodo.

```java
public static String doPut(String urlStr, String jsonBody) throws IOException {
    URL url = new URL(urlStr);
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();

    conn.setRequestMethod("PUT");
    conn.setRequestProperty("Content-Type", "application/json; charset=UTF-8");
    conn.setRequestProperty("Accept", "application/json");
    conn.setDoOutput(true);
    conn.setConnectTimeout(5000);
    conn.setReadTimeout(5000);

    try (OutputStream os = conn.getOutputStream()) {
        byte[] input = jsonBody.getBytes(StandardCharsets.UTF_8);
        os.write(input, 0, input.length);
    }

    int status = conn.getResponseCode();
    String body = (status >= 200 && status < 300)
        ? new String(conn.getInputStream().readAllBytes(), StandardCharsets.UTF_8)
        : new String(conn.getErrorStream().readAllBytes(), StandardCharsets.UTF_8);

    conn.disconnect();
    return body;
}
```

Uso tipico para actualizar un recurso:

```java
String json = """
    {
        "id": 1,
        "nombre": "Juan Perez Actualizado",
        "email": "juan.nuevo@example.com",
        "edad": 31
    }
    """;
String respuesta = doPut("https://api.example.com/usuarios/1", json);
```

## DELETE

DELETE es semanticamente similar a GET en cuanto a que tipicamente no tiene cuerpo, aunque HTTP permite cuerpo en DELETE. Se configura con `setRequestMethod("DELETE")` y no requiere `setDoOutput(true)` a menos que se envie cuerpo.

```java
public static int doDelete(String urlStr) throws IOException {
    URL url = new URL(urlStr);
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();

    conn.setRequestMethod("DELETE");
    conn.setRequestProperty("Authorization", "Bearer token123");
    conn.setConnectTimeout(5000);
    conn.setReadTimeout(5000);

    int status = conn.getResponseCode();

    conn.disconnect();
    return status;
}
```

```java
int statusCode = doDelete("https://api.example.com/usuarios/42");
if (statusCode == 204) {
    System.out.println("Recurso eliminado correctamente");
}
```

## Manejo de cabeceras comunes

Independientemente del metodo, ciertas cabeceras son omnipresentes en las API REST modernas:

- `Content-Type` -- indica el formato del cuerpo de la peticion (`application/json`, `application/xml`, `text/plain`).
- `Accept` -- indica el formato que el cliente acepta en la respuesta.
- `Authorization` -- porta credenciales de autenticacion (Bearer token, Basic auth).
- `User-Agent` -- identifica el cliente que realiza la peticion.
- `Cache-Control` -- controla el comportamiento de cache.

```java
conn.setRequestProperty("Cache-Control", "no-cache");
conn.setRequestProperty("Authorization", "Bearer " + token);
```

## Manejo de redirecciones

Por defecto, `HttpURLConnection` sigue automaticamente las redirecciones HTTP 301, 302, 303 y 307. Si se necesita control manual, se desactiva el seguimiento automatico y se lee la cabecera `Location`:

```java
conn.setInstanceFollowRedirects(false);

int status = conn.getResponseCode();
if (status == 301 || status == 302) {
    String newUrl = conn.getHeaderField("Location");
    System.out.println("Redirigir a: " + newUrl);
}
```
