# Respuestas con HttpResponse

Cuando se ejecuta una peticion, `HttpClient` devuelve un `HttpResponse<T>`, donde `T` es el tipo del cuerpo procesado. La respuesta encapsula el codigo de estado, los headers, la URI final (tras redirecciones), la version del protocolo y el cuerpo transformado segun el `BodyHandler` especificado.

## BodyHandlers

El `BodyHandler` determina como se consume el cuerpo de la respuesta. Se pasa como segundo argumento a `send()` o `sendAsync()` y transforma los bytes entrantes en el tipo deseado. `HttpResponse.BodyHandlers` proporciona implementaciones predefinidas:

- `ofString()`: cuerpo como `String` (UTF-8). Acepta un `Charset` opcional.
- `ofString(Charset)`: cuerpo como String con codificacion explicita.
- `ofByteArray()`: cuerpo como `byte[]`.
- `ofFile(Path)`: escribe el cuerpo directamente en un archivo. Devuelve `Path`.
- `ofInputStream()`: cuerpo como `InputStream` para lectura bajo demanda.
- `ofLines()`: cuerpo como `Stream<String>` (una linea por elemento).
- `ofByteArrayConsumer(Consumer<Optional<byte[]>>)`: consumidor reactivo para cada fragmento.
- `discarding()`: descarta el cuerpo. Devuelve `Void`.
- `buffering(BodyHandler, int)`: envoltura que almacena en buffer antes de pasar al handler real.

```java
import java.net.http.HttpResponse.BodyHandlers;

// Como String
HttpResponse<String> respStr = client.send(request, BodyHandlers.ofString());

// Como String con codificacion
HttpResponse<String> respUtf8 = client.send(request, BodyHandlers.ofString(StandardCharsets.UTF_8));

// Como bytes
HttpResponse<byte[]> respBytes = client.send(request, BodyHandlers.ofByteArray());

// A archivo
Path destino = Path.of("/tmp/descarga.pdf");
HttpResponse<Path> respFile = client.send(request, BodyHandlers.ofFile(destino));

// Como InputStream (lectura perezosa)
HttpResponse<InputStream> respStream = client.send(request, BodyHandlers.ofInputStream());
try (InputStream is = respStream.body()) {
    is.readAllBytes();
}

// Como Stream de lineas
HttpResponse<Stream<String>> respLines = client.send(request, BodyHandlers.ofLines());
respLines.body().forEach(System.out::println);
```

## Buffering

`BodyHandlers.buffering()` envuelve otro `BodyHandler` y almacena en buffer los datos antes de pasarlos al handler subyacente. Esto es util cuando el handler final no puede manejar fragmentos pequeños (por ejemplo, `ofFile()` con disco lento) o cuando se quiere reducir la fragmentacion.

```java
HttpResponse<Path> response = client.send(request,
        BodyHandlers.buffering(BodyHandlers.ofFile(destino), 8192));
```

## Status code y metadatos

La respuesta ofrece acceso directo al codigo de estado y a los headers:

```java
int status = response.statusCode();
HttpHeaders headers = response.headers();
String contentType = headers.firstValue("Content-Type").orElse("unknown");
```

`HttpHeaders` proporciona metodos como `firstValue(String)`, `allValues(String)`, `map()` para inspeccionar y recorrer las cabeceras de forma inmutable.

## URI final y version

Tras las redirecciones, la URI final puede diferir de la URI original. `response.uri()` devuelve la URI real desde la que se obtuvo la respuesta. `response.version()` indica la version del protocolo utilizada (HTTP_1_1 o HTTP_2).

```java
System.out.println("URI final: " + response.uri());
System.out.println("Version: " + response.version());
```

## Ejemplo completo

Un programa que obtiene datos de una API JSON, descarga un archivo adjunto y maneja errores:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpResponse.BodyHandlers;
import java.nio.file.Path;
import java.time.Duration;

public class EjemploHttpResponse {

    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .followRedirects(HttpClient.Redirect.NORMAL)
                .build();

        // Peticion GET a una API
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://jsonplaceholder.typicode.com/posts/1"))
                .header("Accept", "application/json")
                .timeout(Duration.ofSeconds(15))
                .GET()
                .build();

        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

        System.out.println("=== Metadatos ===");
        System.out.println("Status code: " + response.statusCode());
        System.out.println("Protocolo: " + response.version());
        System.out.println("URI final: " + response.uri());

        System.out.println("\n=== Headers ===");
        response.headers().map().forEach((key, values) ->
                System.out.println(key + ": " + String.join(", ", values)));

        System.out.println("\n=== Cuerpo ===");
        if (response.statusCode() == 200) {
            System.out.println(response.body());
        } else {
            System.err.println("Error: " + response.statusCode());
        }

        // Descarga de un recurso a archivo
        HttpRequest downloadReq = HttpRequest.newBuilder()
                .uri(URI.create("https://via.placeholder.com/150.png"))
                .GET()
                .build();

        HttpResponse<Path> fileResponse = client.send(downloadReq,
                BodyHandlers.ofFile(Path.of("imagen.png")));

        System.out.println("\nArchivo descargado en: " + fileResponse.body());
    }
}
```

> El `body()` de `HttpResponse` solo puede llamarse una vez para los tipos `InputStream` y `Stream<String>`, ya que el flujo subyacente se consume. Para `String` y `byte[]` puede llamarse tantas veces como se desee porque el contenido ya esta en memoria.
