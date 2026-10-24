# Async con HttpClient

El modelo asincrono de `HttpClient` permite lanzar peticiones HTTP sin bloquear el hilo de ejecucion, utilizando `CompletableFuture` como mecanismo de composicion. Esto no solo mejora la escalabilidad al liberar hilos mientras se espera la respuesta, sino que permite encadenar transformaciones, combinar resultados de multiples peticiones y manejar errores de forma declarativa.

## sendAsync vs send

La diferencia fundamental entre `send` y `sendAsync` es el modelo de ejecucion:

- `send()`: metodo sincrono y bloqueante. El hilo actual se detiene hasta recibir la respuesta completa. Adecuado para scripts, prototipos o cuando la concurrencia no es necesaria.

- `sendAsync()`: metodo no bloqueante. Devuelve inmediatamente un `CompletableFuture<HttpResponse<T>>`. El hilo actual continua con otras tareas mientras el cliente HTTP espera la respuesta en segundo plano.

```java
// Sincrono (bloquea)
HttpResponse<String> syncResp = client.send(request, BodyHandlers.ofString());

// Asincrono (no bloquea)
CompletableFuture<HttpResponse<String>> asyncResp = client.sendAsync(request, BodyHandlers.ofString());
```

## Completar con thenApply y thenAccept

`CompletableFuture` permite registrar callbacks que se ejecutan cuando la respuesta llega. `thenApply` transforma el resultado; `thenAccept` lo consume sin devolver valor.

```java
client.sendAsync(request, BodyHandlers.ofString())
       .thenApply(HttpResponse::body)              // extrae el body
       .thenAccept(body -> System.out.println(body)) // lo consume
       .join(); // espera a que termine (opcional)
```

`thenApply` devuelve un nuevo `CompletableFuture<U>`, lo que permite encadenar mas transformaciones. `thenAccept` devuelve `CompletableFuture<Void>` y es util para efectos secundarios.

## Manejo de errores con exceptionally

Si la peticion falla (timeout, error de red, 5xx que se desea tratar como error), el `CompletableFuture` se completa excepcionalmente. `exceptionally` permite recuperar el error y devolver un valor por defecto.

```java
CompletableFuture<String> future = client
        .sendAsync(request, BodyHandlers.ofString())
        .thenApply(HttpResponse::body)
        .exceptionally(throwable -> {
            System.err.println("Error: " + throwable.getMessage());
            return "{}"; // JSON vacio como fallback
        });

String resultado = future.join();
```

Tambien se puede usar `handle` para manejar tanto exito como error en un mismo callback:

```java
CompletableFuture<String> future = client
        .sendAsync(request, BodyHandlers.ofString())
        .thenApply(HttpResponse::body)
        .handle((body, throwable) -> {
            if (throwable != null) {
                System.err.println("Error: " + throwable.getMessage());
                return "{}";
            }
            return body;
        });
```

## Composicion con thenCompose y thenCombine

`thenCompose` (equivalente a `flatMap`) es util cuando una peticion depende del resultado de otra. Permite encadenar peticiones secuencialmente sin anidar callbacks.

```java
// Primero obtener el token, luego usarlo
CompletableFuture<HttpResponse<String>> loginFuture = client
        .sendAsync(loginRequest, BodyHandlers.ofString());

CompletableFuture<HttpResponse<String>> dataFuture = loginFuture
        .thenCompose(loginResp -> {
            String token = extraerToken(loginResp.body());
            HttpRequest dataReq = HttpRequest.newBuilder()
                    .uri(URI.create("https://api.example.com/data"))
                    .header("Authorization", "Bearer " + token)
                    .GET()
                    .build();
            return client.sendAsync(dataReq, BodyHandlers.ofString());
        });

dataFuture.thenAccept(resp -> System.out.println(resp.body())).join();
```

`thenCombine` combina los resultados de dos peticiones independientes cuando ambas completan.

```java
CompletableFuture<HttpResponse<String>> future1 =
        client.sendAsync(request1, BodyHandlers.ofString());
CompletableFuture<HttpResponse<String>> future2 =
        client.sendAsync(request2, BodyHandlers.ofString());

CompletableFuture<String> combined = future1
        .thenCombine(future2, (resp1, resp2) -> {
            return "Respuesta 1: " + resp1.body() + "\nRespuesta 2: " + resp2.body();
        });

System.out.println(combined.join());
```

## Multiples peticiones paralelas con allOf

`CompletableFuture.allOf` permite lanzar varias peticiones en paralelo y esperar a que todas completen. Devuelve un `CompletableFuture<Void>` que se completa cuando todos los futures dados completan.

```java
List<HttpRequest> requests = List.of(
        HttpRequest.newBuilder().uri(URI.create("https://api.example.com/a")).GET().build(),
        HttpRequest.newBuilder().uri(URI.create("https://api.example.com/b")).GET().build(),
        HttpRequest.newBuilder().uri(URI.create("https://api.example.com/c")).GET().build()
);

List<CompletableFuture<HttpResponse<String>>> futures = requests.stream()
        .map(req -> client.sendAsync(req, BodyHandlers.ofString()))
        .toList();

CompletableFuture<Void> allDone = CompletableFuture.allOf(
        futures.toArray(new CompletableFuture[0]));

CompletableFuture<List<String>> allBodies = allDone
        .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .map(HttpResponse::body)
                .toList());

List<String> bodies = allBodies.join();
bodies.forEach(System.out::println);
```

> `allOf` devuelve un `CompletableFuture<Void>`, por lo que no se puede acceder directamente a los resultados. Es necesario mapear sobre la lista original de futures una vez que `allOf` completa, usando `join()` ya en ese punto no bloquea porque todos los futures ya estan completados.

## Ejemplo completo

Un cliente que consulta tres endpoints en paralelo, transforma los resultados, los combina y maneja errores:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpResponse.BodyHandlers;
import java.time.Duration;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import java.util.stream.Collectors;

public class AsyncHttpExample {

    public static void main(String[] args) {
        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .executor(Executors.newFixedThreadPool(5))
                .build();

        List<String> urls = List.of(
                "https://jsonplaceholder.typicode.com/posts/1",
                "https://jsonplaceholder.typicode.com/posts/2",
                "https://jsonplaceholder.typicode.com/posts/3"
        );

        List<CompletableFuture<HttpResponse<String>>> futures = urls.stream()
                .map(url -> {
                    HttpRequest request = HttpRequest.newBuilder()
                            .uri(URI.create(url))
                            .timeout(Duration.ofSeconds(15))
                            .GET()
                            .build();
                    return client.sendAsync(request, BodyHandlers.ofString());
                })
                .toList();

        CompletableFuture<Void> allFutures = CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0]));

        CompletableFuture<List<String>> result = allFutures
                .thenApply(ignored -> futures.stream()
                        .map(f -> {
                            try {
                                HttpResponse<String> resp = f.join();
                                if (resp.statusCode() == 200) {
                                    return resp.body();
                                } else {
                                    return "Error HTTP: " + resp.statusCode();
                                }
                            } catch (Exception e) {
                                return "Excepcion: " + e.getMessage();
                            }
                        })
                        .toList())
                .exceptionally(ex -> List.of("Fallo general: " + ex.getMessage()));

        List<String> bodies = result.join();
        bodies.forEach(body -> {
            System.out.println("--- Respuesta ---");
            System.out.println(body);
        });

        System.out.println("\nTodas las peticiones completadas.");
    }
}
```

```java
// Ejemplo con thenCompose: cadena de peticiones dependientes
CompletableFuture<String> postTitleFuture = client
        .sendAsync(
                HttpRequest.newBuilder()
                        .uri(URI.create("https://jsonplaceholder.typicode.com/posts/1"))
                        .GET().build(),
                BodyHandlers.ofString())
        .thenCompose(resp -> {
            // Extraer userId del JSON (simplificado)
            String body = resp.body();
            String userId = body.replaceAll(".*\"userId\": (\\d+).*", "$1");

            HttpRequest userReq = HttpRequest.newBuilder()
                    .uri(URI.create("https://jsonplaceholder.typicode.com/users/" + userId))
                    .GET().build();

            return client.sendAsync(userReq, BodyHandlers.ofString());
        })
        .thenApply(HttpResponse::body)
        .exceptionally(ex -> "{\"name\": \"desconocido\"}");

System.out.println("Usuario del post: " + postTitleFuture.join());
```

> El uso intensivo de `sendAsync` sin un `Executor` personalizado puede saturar el *common fork-join pool*. Para aplicaciones que lanzan decenas o cientos de peticiones concurrentes, es obligatorio configurar un `Executor` con el tamano de pool adecuado y apagarlo correctamente al finalizar.
