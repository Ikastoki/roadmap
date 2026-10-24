# Introduccion a HttpClient

Hasta Java 11, la unica forma nativa de realizar peticiones HTTP en Java era a traves de `HttpURLConnection`, una clase disenada en los anos 90 que arrastraba un diseno rigido, una API verbosa y una gestion de conexiones que exigia un esfuerzo considerable para tareas modernas como peticiones asincronas, streaming eficiente o soporte de HTTP/2. La clase `java.net.http.HttpClient`, introducida como incubadora en Java 9 y estandarizada en Java 11, vino a cubrir ese vacio con una API funcional, fluida y preparada para los protocolos actuales.

## Diseno y arquitectura

`HttpClient` no es una simple envoltura sobre `HttpURLConnection`: es una reimplementacion completa que opera sobre el nuevo modulo `jdk.incubator.http` (luego `java.net.http`) y se apoya en `Selector` y `SocketChannel` de NIO para gestionar conexiones no bloqueantes de forma nativa. Esto permite que una misma instancia de `HttpClient` reutilice conexiones, mantenga un pool de sockets y maneje multiples peticiones concurrentes sin necesidad de crear hilos manualmente.

La clase sigue el patron *builder*: no se instancia directamente con `new`, sino a traves de `HttpClient.newBuilder()`. Sus metodos devuelven el propio builder, lo que permite encadenar configuraciones como version del protocolo, timeout, politicas de redireccion o el executor para tareas asincronas.

```java
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(10))
        .build();
```

## Soporte de protocolos

`HttpClient` soporta HTTP/1.1 y HTTP/2. Cuando se configura con `HTTP_2` (valor por defecto), el cliente intenta negociar HTTP/2 mediante ALPN. Si el servidor no soporta HTTP/2, la conexion degrada automaticamente a HTTP/1.1 sin que el desarrollador deba intervenir. Tambien admite HTTPS a traves de `SSLContext` personalizado y websockets mediante `WebSocket.Builder`.

## Operacion sincrona y asincrona

El cliente ofrece dos modos de envio:

- **Sincrono** con `send(request, bodyHandler)`: bloquea el hilo actual hasta recibir la respuesta completa. Es el modelo mas simple, adecuado para scripts, procesos secuenciales o entornos donde el bloqueo no es un problema.

- **Asincrono** con `sendAsync(request, bodyHandler)`: devuelve un `CompletableFuture<HttpResponse<T>>` que se completa cuando llega la respuesta. Esto permite lanzar multiples peticiones en paralelo y componer resultados sin gestionar hilos manualmente.

```java
// Llamada sincrona
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// Llamada asincrona
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
      .thenAccept(resp -> System.out.println(resp.body()));
```

> Aunque `sendAsync` no bloquea, el `CompletableFuture` se ejecuta sobre el *common fork-join pool* a menos que se especifique un `Executor` en el builder. Para aplicaciones de alto rendimiento conviene proporcionar un pool personalizado.

## Ventajas sobre HttpURLConnection

`HttpURLConnection` requeria configurar manualmente cada aspecto de la conexion: abrir la conexion, configurar propiedades, escribir el cuerpo, leer la respuesta y gestionar excepciones, todo con multiples pasos y codigo verboso. Ademas, su modelo de bloqueo obligaba a crear hilos para concurrencia y no tenia soporte nativo de HTTP/2.

`HttpClient` unifica todo en una API fluida: una sola llamada `send` o `sendAsync` encapsula la conexion, el envio de la peticion, la gestion de headers y la lectura de la respuesta. La reutilizacion de conexiones es automatica, las redirecciones se manejan con una configuracion, y la asincronia llega de fabrica con `CompletableFuture`.

| Aspecto | HttpURLConnection | HttpClient |
|---|---|---|
| API | Verbosa, basada en setters | Fluida, builder pattern |
| HTTP/2 | No (solo 1.1) | Si, con degradacion |
| Asincronia | Manual (hilos) | Nativo (CompletableFuture) |
| Streaming | Limitado | BodyHandlers/Subscribers |
| Conexiones | Una por instancia | Pool reutilizable |

```java
// HttpURLConnection: fragmento tipico
URL url = new URL("https://api.example.com");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setRequestProperty("Accept", "application/json");
int status = conn.getResponseCode();
BufferedReader in = new BufferedReader(new InputStreamReader(conn.getInputStream()));

// HttpClient: equivalente moderno
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com"))
        .header("Accept", "application/json")
        .build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```
