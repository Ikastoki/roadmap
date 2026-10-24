# CompletableFuture para redes

`CompletableFuture` es la piedra angular de la programacion asincrona en Java moderno. Introducido en Java 8 como una evolucion de `Future`, permite componer operaciones asincronas mediante un DSL funcional que elimina la complejidad de los callbacks anidados. En el contexto de redes, `CompletableFuture` permite realizar peticiones HTTP, consultas a bases de datos o llamadas a servicios remotos sin bloquear hilos, y combinar sus resultados de forma elegante.

## supplyAsync: tareas asincronas

`supplyAsync` ejecuta una tarea en un hilo del pool comun de ForkJoinPool (o en un `Executor` personalizado) y devuelve un `CompletableFuture` que se completara con el resultado de la operacion.

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        URL url = new URL("https://api.example.com/data");
        return new String(url.openStream().readAllBytes(), StandardCharsets.UTF_8);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
});
```

Por defecto usa `ForkJoinPool.commonPool()`. Para aplicaciones de red con multiples peticiones simultaneas, es recomendable proporcionar un `Executor` con un pool de hilos dimensionado adecuadamente.

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchUrl("https://api.example.com"), pool);
```

## ThenApply, thenAccept y exceptionally

El encadenamiento es donde `CompletableFuture` muestra su potencia. Cada etapa recibe el resultado de la anterior y puede transformarlo, consumirlo o manejar errores.

```java
CompletableFuture.supplyAsync(() -> fetchUrl("https://api.example.com/user"))
    .thenApply(json -> parseUser(json))           // transforma String -> User
    .thenAccept(user -> System.out.println(user)) // consume el resultado
    .exceptionally(ex -> {
        System.err.println("Error: " + ex.getMessage());
        return null; // valor por defecto ante fallo
    });
```

`thenApply` transforma el resultado (mapeo), `thenAccept` lo consume sin devolver valor, y `exceptionally` proporciona un valor alternativo si ocurre una excepcion en cualquier etapa anterior.

## Combinacion de multiples futures

### allOf: esperar todos

`CompletableFuture.allOf` toma un array de `CompletableFuture` y devuelve uno que se completa cuando todos se han completado. Es util para lanzar varias peticiones en paralelo y esperar a que terminen todas.

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> fetchUrl("https://api1.com"));
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> fetchUrl("https://api2.com"));
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> fetchUrl("https://api3.com"));

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> {
    String r1 = f1.join(); // join no lanza checked exceptions
    String r2 = f2.join();
    String r3 = f3.join();
    System.out.println(r1 + r2 + r3);
});
```

`join()` es similar a `get()` pero lanza `CompletionException` unchecked en lugar de `ExecutionException`, simplificando el manejo de errores en cadenas de futures.

### anyOf: el primero que termine

`CompletableFuture.anyOf` devuelve un future que se completa con el resultado del primer future en completarse (exitosamente o con error). Ideal para fallback: lanzar la misma peticion a varios servidores y usar la primera respuesta.

```java
CompletableFuture<String> servidor1 = CompletableFuture.supplyAsync(() -> fetchUrl("https://backup1.com/api"));
CompletableFuture<String> servidor2 = CompletableFuture.supplyAsync(() -> fetchUrl("https://backup2.com/api"));

CompletableFuture<Object> primero = CompletableFuture.anyOf(servidor1, servidor2);
primero.thenAccept(respuesta -> System.out.println("Respuesta: " + respuesta));
```

### thenCombine: combinar dos resultados

`thenCombine` fusiona los resultados de dos futures independientes mediante una funcion `BiFunction`.

```java
CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> fetchUrl("https://api.example.com/user"));
CompletableFuture<String> postsFuture = CompletableFuture.supplyAsync(() -> fetchUrl("https://api.example.com/posts"));

userFuture.thenCombine(postsFuture, (user, posts) -> {
    return "Usuario: " + user + "\nPosts: " + posts;
}).thenAccept(System.out::println);
```

## Encadenamiento completo: ejemplo real

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ClienteAsincrono {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(5);
        HttpClient client = HttpClient.newBuilder().executor(pool).build();

        CompletableFuture<String> api1 = CompletableFuture.supplyAsync(() -> {
            HttpRequest req = HttpRequest.newBuilder()
                .uri(URI.create("https://jsonplaceholder.typicode.com/todos/1"))
                .GET().build();
            try {
                return client.send(req, HttpResponse.BodyHandlers.ofString()).body();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });

        CompletableFuture<String> api2 = CompletableFuture.supplyAsync(() -> {
            HttpRequest req = HttpRequest.newBuilder()
                .uri(URI.create("https://jsonplaceholder.typicode.com/posts/1"))
                .GET().build();
            try {
                return client.send(req, HttpResponse.BodyHandlers.ofString()).body();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });

        api1.thenCombine(api2, (a, b) -> "Resultado combinado:\n" + a + "\n---\n" + b)
            .thenAccept(System.out::println)
            .exceptionally(ex -> {
                System.err.println("Fallo: " + ex.getMessage());
                return null;
            })
            .thenRun(() -> pool.shutdown());
    }
}
```

> `CompletableFuture` sigue el patron Promise/Future. Una vez completado (exitosamente o con error), su estado es inmutable. Todos los metodos de encadenamiento devuelven nuevos `CompletableFuture`, no mutan el original. Esto permite composicion segura y reutilizacion de resultados parciales.
