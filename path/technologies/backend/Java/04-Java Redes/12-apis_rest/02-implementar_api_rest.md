# Implementar API REST basica

Java proporciona desde la version 6 un servidor HTTP ligero en el paquete `com.sun.net.httpserver` que permite implementar APIs REST sin necesidad de frameworks externos como Spring Boot o Jersey. Aunque no es adecuado para produccion de alto volumen (carece de soporte asincrono nativo avanzado y de configuraciones de tuning fino), resulta ideal para prototipos, herramientas internas y aprendizaje, ya que expone directamente el modelo peticion-respuesta de HTTP.

## HttpServer

La clase `HttpServer` gestiona el ciclo de vida del servidor: escucha en un puerto, acepta conexiones y las enruta a los manejadores registrados. Se crea mediante el metodo factory `HttpServer.create()`.

```java
import com.sun.net.httpserver.HttpServer;
import java.net.InetSocketAddress;
import java.util.concurrent.Executors;

HttpServer server = HttpServer.create(
    new InetSocketAddress(8080), 0);
server.setExecutor(Executors.newFixedThreadPool(4));
server.start();
```

El segundo parametro de `create()` es el `backlog`: el numero maximo de conexiones entrantes en cola. Un valor de 0 usa el valor por defecto del sistema operativo.

> `HttpServer` pertenece al paquete `com.sun.net.httpserver`, que no forma parte de la especificacion oficial de Java SE, aunque esta presente en todas las implementaciones de Oracle/OpenJDK. Para entornos donde se requiera portabilidad total, es preferible usar Jakarta EE o un framework embebido como Jetty o Undertow.

## HttpContext y HttpHandler

Cada ruta del servidor se registra como un `HttpContext` asociado a un `HttpHandler`. El metodo `handle(HttpExchange exchange)` recibe el intercambio completo y debe producir una respuesta.

```java
server.createContext("/api/usuarios", new UsuariosHandler());

static class UsuariosHandler implements HttpHandler {
    @Override
    public void handle(HttpExchange exchange) throws IOException {
        // ...
    }
}
```

Un solo contexto puede manejar todos los metodos HTTP; la discriminacion se hace dentro del handler.

## HttpExchange

`HttpExchange` encapsula la peticion y la respuesta. Proporciona acceso a todos los elementos de la comunicacion HTTP.

### Peticion entrante

```java
String metodo = exchange.getRequestMethod();               // GET, POST, PUT, DELETE...
URI uri = exchange.getRequestURI();                        // /api/usuarios?page=1
String path = uri.getPath();                               // /api/usuarios
String query = uri.getQuery();                             // page=1
Headers headers = exchange.getRequestHeaders();            // cabeceras de peticion
InputStream body = exchange.getRequestBody();              // cuerpo para leer
```

### Respuesta saliente

```java
Headers responseHeaders = exchange.getResponseHeaders();
responseHeaders.set("Content-Type", "application/json");

// Enviar cabeceras con codigo de estado y longitud (-1 si es chunked)
exchange.sendResponseHeaders(200, responseBytes.length);

OutputStream os = exchange.getResponseBody();
os.write(responseBytes);
os.close();
```

> `sendResponseHeaders()` debe llamarse **antes** de escribir el cuerpo. Una vez llamada, no se pueden modificar las cabeceras. La longitud del cuerpo debe conocerse de antemano; si no es posible, se pasa -1 para habilitar transferencia chunked.

## Ejemplo completo: servidor REST minimo

```java
import com.sun.net.httpserver.*;
import java.io.*;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.Executors;

public class ServidorRESTMinimo {

    public static void main(String[] args) throws IOException {
        HttpServer server = HttpServer.create(
            new InetSocketAddress(8080), 0);

        server.createContext("/api/saludo", new SaludoHandler());
        server.createContext("/api/echo", new EchoHandler());

        server.setExecutor(Executors.newCachedThreadPool());
        server.start();

        System.out.println("Servidor iniciado en http://localhost:8080");
    }

    static class SaludoHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            if (!"GET".equalsIgnoreCase(exchange.getRequestMethod())) {
                exchange.sendResponseHeaders(405, -1);
                return;
            }
            byte[] resp = "{\"mensaje\":\"Hola mundo\"}"
                    .getBytes(StandardCharsets.UTF_8);
            exchange.getResponseHeaders().set("Content-Type",
                    "application/json; charset=UTF-8");
            exchange.sendResponseHeaders(200, resp.length);
            exchange.getResponseBody().write(resp);
            exchange.close();
        }
    }

    static class EchoHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            // Lee el cuerpo y lo devuelve como respuesta
            String body = new String(exchange.getRequestBody()
                    .readAllBytes(), StandardCharsets.UTF_8);
            byte[] resp = body.getBytes(StandardCharsets.UTF_8);
            exchange.getResponseHeaders().set("Content-Type",
                    exchange.getRequestHeaders()
                            .getFirst("Content-Type"));
            exchange.sendResponseHeaders(200, resp.length);
            exchange.getResponseBody().write(resp);
            exchange.close();
        }
    }
}
```

## Cierre del servidor

El servidor se detiene con `server.stop(delay)`, donde `delay` es el numero de segundos de espera para que las peticiones en curso finalicen antes de cerrar los sockets.

```java
server.stop(2); // espera 2 segundos
```

## Limitaciones

`com.sun.net.httpserver` no soporta de forma nativa:
- Enrutamiento por expresiones regulares o parametros de path (`/api/usuarios/{id}`).
- Content negotiation automatica.
- Soporte asincrono completo (AIO).
- TLS/SSL directo, aunque puede configurarse con `HttpsServer`.

Para aplicaciones que requieran estas capacidades, se recomienda migrar a Spring Boot, JAX-RS (Jersey/RESTEasy) o frameworks reactivos como Vert.x.
