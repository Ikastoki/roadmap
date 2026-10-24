# Ejemplo completo: API CRUD

Para ilustrar todos los conceptos de una API REST en Java con la API `com.sun.net.httpserver`, se presenta una implementacion completa de un CRUD de usuarios. El almacenamiento se realiza en memoria mediante `ConcurrentHashMap` y la serializacion se delega en Gson, la biblioteca de Google para JSON.

## Estructura del proyecto

```
src/
  Usuario.java           -- modelo de datos
  UsuarioRepository.java -- capa de acceso a datos (memoria)
  UsuarioController.java -- manejador HTTP
  ServidorCRUD.java      -- punto de entrada
```

## Dependencia: Gson

Gson debe estar en el classpath. Si se usa Maven, se anade:

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.10.1</version>
</dependency>
```

## Modelo: Usuario

```java
import java.util.Objects;

public class Usuario {
    private Long id;
    private String nombre;
    private String email;

    public Usuario() {}

    public Usuario(Long id, String nombre, String email) {
        this.id = id;
        this.nombre = nombre;
        this.email = email;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Usuario usuario)) return false;
        return Objects.equals(id, usuario.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

## Repositorio

```java
import java.util.Collection;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class UsuarioRepository {
    private final ConcurrentHashMap<Long, Usuario> almacen
            = new ConcurrentHashMap<>();
    private final AtomicLong generadorId = new AtomicLong(1);

    public Collection<Usuario> listar() {
        return almacen.values();
    }

    public Usuario obtener(Long id) {
        return almacen.get(id);
    }

    public Usuario crear(Usuario usuario) {
        Long id = generadorId.getAndIncrement();
        usuario.setId(id);
        almacen.put(id, usuario);
        return usuario;
    }

    public Usuario actualizar(Long id, Usuario usuario) {
        usuario.setId(id);
        Usuario previo = almacen.replace(id, usuario);
        return previo != null ? usuario : null;
    }

    public boolean eliminar(Long id) {
        return almacen.remove(id) != null;
    }
}
```

> `ConcurrentHashMap` es seguro para entornos multihilo sin necesidad de sincronizacion externa, siempre que las operaciones compuestas (como `replace` y `remove`) se usen correctamente. Un `AtomicLong` garantiza que los IDs sean unicos aunque varios hilos creen usuarios simultaneamente.

## Controlador

El controlador parsea la URI para extraer parametros de path, delega en el repositorio y serializa la respuesta con Gson.

```java
import com.google.gson.Gson;
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import java.io.*;
import java.net.URI;
import java.nio.charset.StandardCharsets;

public class UsuarioController implements HttpHandler {
    private final UsuarioRepository repo = new UsuarioRepository();
    private final Gson gson = new Gson();

    @Override
    public void handle(HttpExchange exchange) throws IOException {
        try {
            String metodo = exchange.getRequestMethod().toUpperCase();
            String path = exchange.getRequestURI().getPath();
            // /api/usuarios o /api/usuarios/{id}

            if (path.equals("/api/usuarios")) {
                switch (metodo) {
                    case "GET"  -> handleListar(exchange);
                    case "POST" -> handleCrear(exchange);
                    default     -> enviarError(exchange, 405, "Metodo no permitido");
                }
            } else if (path.matches("/api/usuarios/\\d+")) {
                Long id = Long.parseLong(
                    path.substring(path.lastIndexOf('/') + 1));
                switch (metodo) {
                    case "GET"    -> handleObtener(exchange, id);
                    case "PUT"    -> handleActualizar(exchange, id);
                    case "DELETE" -> handleEliminar(exchange, id);
                    default       -> enviarError(exchange, 405, "Metodo no permitido");
                }
            } else {
                enviarError(exchange, 404, "Recurso no encontrado");
            }
        } catch (Exception e) {
            enviarError(exchange, 500, "Error interno: " + e.getMessage());
        }
    }

    private void handleListar(HttpExchange exchange) throws IOException {
        String json = gson.toJson(repo.listar());
        enviarRespuesta(exchange, 200, json);
    }

    private void handleObtener(HttpExchange exchange, Long id)
            throws IOException {
        Usuario u = repo.obtener(id);
        if (u == null) {
            enviarError(exchange, 404, "Usuario no encontrado");
            return;
        }
        enviarRespuesta(exchange, 200, gson.toJson(u));
    }

    private void handleCrear(HttpExchange exchange) throws IOException {
        Usuario u = gson.fromJson(leerCuerpo(exchange), Usuario.class);
        if (u.getNombre() == null || u.getEmail() == null) {
            enviarError(exchange, 400,
                    "nombre y email son obligatorios");
            return;
        }
        Usuario creado = repo.crear(u);
        String json = gson.toJson(creado);
        enviarRespuesta(exchange, 201, json);
    }

    private void handleActualizar(HttpExchange exchange, Long id)
            throws IOException {
        Usuario u = gson.fromJson(leerCuerpo(exchange), Usuario.class);
        Usuario actualizado = repo.actualizar(id, u);
        if (actualizado == null) {
            enviarError(exchange, 404, "Usuario no encontrado");
            return;
        }
        enviarRespuesta(exchange, 200, gson.toJson(actualizado));
    }

    private void handleEliminar(HttpExchange exchange, Long id)
            throws IOException {
        if (repo.eliminar(id)) {
            exchange.sendResponseHeaders(204, -1);
        } else {
            enviarError(exchange, 404, "Usuario no encontrado");
        }
        exchange.close();
    }

    private String leerCuerpo(HttpExchange exchange) throws IOException {
        return new String(exchange.getRequestBody()
                .readAllBytes(), StandardCharsets.UTF_8);
    }

    private void enviarRespuesta(HttpExchange exchange, int codigo,
            String json) throws IOException {
        byte[] bytes = json.getBytes(StandardCharsets.UTF_8);
        exchange.getResponseHeaders().set("Content-Type",
                "application/json; charset=UTF-8");
        exchange.sendResponseHeaders(codigo, bytes.length);
        exchange.getResponseBody().write(bytes);
        exchange.close();
    }

    private void enviarError(HttpExchange exchange, int codigo,
            String mensaje) throws IOException {
        String json = gson.toJson(
                java.util.Map.of("error", mensaje));
        byte[] bytes = json.getBytes(StandardCharsets.UTF_8);
        exchange.getResponseHeaders().set("Content-Type",
                "application/json; charset=UTF-8");
        exchange.sendResponseHeaders(codigo, bytes.length);
        exchange.getResponseBody().write(bytes);
        exchange.close();
    }
}
```

## Servidor

```java
import com.sun.net.httpserver.HttpServer;
import java.net.InetSocketAddress;
import java.util.concurrent.Executors;

public class ServidorCRUD {
    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(
                new InetSocketAddress(8080), 0);
        server.createContext("/api/usuarios",
                new UsuarioController());
        server.setExecutor(Executors.newFixedThreadPool(4));
        server.start();
        System.out.println(
            "CRUD disponible en http://localhost:8080/api/usuarios");
    }
}
```

## Prueba con curl

```bash
# Crear usuario
curl -X POST http://localhost:8080/api/usuarios \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Ana","email":"ana@example.com"}'

# Listar usuarios
curl http://localhost:8080/api/usuarios

# Obtener usuario por id
curl http://localhost:8080/api/usuarios/1

# Actualizar usuario
curl -X PUT http://localhost:8080/api/usuarios/1 \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Ana G.","email":"ana@example.com"}'

# Eliminar usuario
curl -X DELETE http://localhost:8080/api/usuarios/1
```

> El controlador utiliza una unica instancia de `UsuarioRepository`, lo que implica que los datos persisten mientras el servidor este en ejecucion. En un escenario real, el repositorio delegaria en una base de datos y se inyectaria como dependencia.
