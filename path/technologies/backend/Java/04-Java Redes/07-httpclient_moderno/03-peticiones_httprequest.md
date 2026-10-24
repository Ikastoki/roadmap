# Peticiones con HttpRequest

Una vez construido el `HttpClient`, el siguiente paso es modelar la peticion. `HttpRequest` representa una solicitud HTTP completa: metodo, URI, headers, cuerpo y opciones adicionales como timeout. Al igual que `HttpClient`, sigue el patron builder mediante `HttpRequest.newBuilder()`.

## HttpRequest.newBuilder()

El builder se obtiene con `HttpRequest.newBuilder()` o `HttpRequest.newBuilder(URI)`. El primero requiere llamar a `.uri()` posteriormente; el segundo establece la URI directamente. Ambas variantes devuelven un `HttpRequest.Builder` que permite encadenar configuraciones.

```java
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .GET()
        .build();
```

## URI y metodo HTTP

La URI se define con `uri(URI)` y es el unico campo obligatorio. El metodo HTTP se establece explicitamente con `GET()`, `POST(BodyPublisher)`, `PUT(BodyPublisher)`, `DELETE()` o `method(String, BodyPublisher)`. Si no se especifica, el valor por defecto es GET.

```java
// GET explicito
HttpRequest get = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/resource"))
        .GET()
        .build();

// POST con cuerpo
HttpRequest post = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/resource"))
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString("{\"name\": \"Juan\"}"))
        .build();

// PUT
HttpRequest put = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/resource/1"))
        .header("Content-Type", "application/json")
        .PUT(HttpRequest.BodyPublishers.ofString("{\"name\": \"Actualizado\"}"))
        .build();

// DELETE
HttpRequest delete = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/resource/1"))
        .DELETE()
        .build();
```

Para metodos menos comunes como PATCH, se usa `method()`:

```java
HttpRequest patch = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/resource/1"))
        .header("Content-Type", "application/json")
        .method("PATCH", HttpRequest.BodyPublishers.ofString("{\"name\": \"Parcial\"}"))
        .build();
```

## Headers

Los headers se anaden con `header(String name, String value)` o multiples con `headers(String... pairs)`. Cada llamada a `header()` anade un nuevo header; si se repite la misma clave, se anade un valor adicional (header multiple).

```java
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com"))
        .header("Authorization", "Bearer token123")
        .header("Accept", "application/json")
        .header("Accept", "application/xml") // dos valores para Accept
        .GET()
        .build();

// Variante con pares clave-valor
HttpRequest request2 = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com"))
        .headers("Content-Type", "application/json",
                 "Accept", "application/json",
                 "Authorization", "Bearer token123")
        .POST(HttpRequest.BodyPublishers.noBody())
        .build();
```

## BodyPublishers

El cuerpo de la peticion se modela mediante un `BodyPublisher`, que implementa `Flow.Publisher<ByteBuffer>` de Reactive Streams. `HttpRequest.BodyPublishers` ofrece metodos estaticos para los casos mas comunes:

- `ofString(String)`: cuerpo a partir de un String, usando UTF-8 por defecto. Acepta un `Charset` opcional.
- `ofByteArray(byte[])`: cuerpo desde un array de bytes. Util para datos binarios.
- `ofFile(Path)`: envia un archivo completo. Util para subidas multipart.
- `ofInputStream(Supplier<InputStream>)`: cuerpo generado bajo demanda desde un InputStream. El supplier se invoca cuando el cuerpo se va a enviar.
- `ofByteArrays(Function<Long, Optional<byte[]>>)`: publicador reactivo para flujos fragmentados.
- `noBody()`: cuerpo vacio. Equivalente a `GET()` o `DELETE()`.
- `concat(BodyPublisher...)`: concatena multiples publicadores secuencialmente.

```java
import java.net.http.HttpRequest.BodyPublishers;

// String
HttpRequest req1 = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/data"))
        .POST(BodyPublishers.ofString("{\"key\": \"value\"}", Charset.forName("UTF-8")))
        .build();

// Byte array
byte[] binaryData = {0x48, 0x65, 0x6C, 0x6C, 0x6F};
HttpRequest req2 = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/upload"))
        .POST(BodyPublishers.ofByteArray(binaryData))
        .build();

// Archivo
Path file = Path.of("/tmp/imagen.png");
HttpRequest req3 = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/upload"))
        .header("Content-Type", "image/png")
        .POST(BodyPublishers.ofFile(file))
        .build();

// InputStream bajo demanda
HttpRequest req4 = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/upload"))
        .POST(BodyPublishers.ofInputStream(() -> new FileInputStream("/tmp/data.bin")))
        .build();

// Sin cuerpo
HttpRequest req5 = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/delete/1"))
        .POST(BodyPublishers.noBody())
        .build();
```

> `BodyPublishers.ofFile(Path)` lee el archivo completo en memoria solo si el contenido es menor que el buffer interno; de lo contrario, lo transmite en chunks. Para archivos muy grandes, combinar con `BodyPublishers.ofInputStream()` puede ser mas eficiente en memoria.

## Timeout

El metodo `timeout(Duration)` establece el tiempo maximo de espera para la respuesta completa. Si transcurre sin recibir respuesta, se lanza `java.net.http.HttpTimeoutException`.

```java
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/slow"))
        .timeout(Duration.ofSeconds(30))
        .GET()
        .build();
```

> El timeout de `HttpRequest` es independiente del `connectTimeout` del `HttpClient`. El primero controla la duracion total de la peticion; el segundo solo la fase de conexion TCP. Si se establecen ambos, el timeout de la peticion deberia ser mayor.

## Version y expectativa continua

`HttpRequest` permite ademas forzar una version de protocolo especifica para una peticion concreta (aunque herede la del cliente) y controlar la cabecera `Expect` mediante `expectContinue(boolean)`. Si se activa `expectContinue(true)`, el cliente envia la cabecera `Expect: 100-continue` y espera la confirmacion del servidor antes de enviar el cuerpo.

```java
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/upload-large"))
        .expectContinue(true)
        .POST(BodyPublishers.ofFile(Path.of("/tmp/large.bin")))
        .build();
```
