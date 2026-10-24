# JSON en Java

JSON (JavaScript Object Notation) es el formato de intercambio de datos predominante en APIs REST modernas. Java carece de un soporte nativo tan fluido como otros lenguajes, pero bibliotecas como Gson y Jackson proporcionan una integracion completa con el sistema de tipos de Java, permitiendo convertir objetos Java a JSON y viceversa con minima configuracion.

## Gson

Gson, desarrollada por Google, es la opcion mas ligera y sencilla para proyectos que no requieren un control exhaustivo sobre la serializacion.

### Serializacion basica

```java
import com.google.gson.Gson;

public class GsonBasico {
    record Usuario(Long id, String nombre, String email) {}

    public static void main(String[] args) {
        Gson gson = new Gson();

        // Objeto a JSON
        Usuario u = new Usuario(1L, "Ana", "ana@example.com");
        String json = gson.toJson(u);
        System.out.println(json);
        // {"id":1,"nombre":"Ana","email":"ana@example.com"}

        // JSON a objeto
        String entrada = "{\"id\":2,\"nombre\":\"Luis\",\"email\":\"luis@e.com\"}";
        Usuario usuario = gson.fromJson(entrada, Usuario.class);
        System.out.println(usuario.nombre()); // Luis
    }
}
```

### Configuracion con GsonBuilder

```java
Gson gson = new GsonBuilder()
        .setPrettyPrinting()
        .serializeNulls()
        .setDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'")
        .disableHtmlEscaping()
        .create();
```

## Jackson

Jackson es la biblioteca mas completa y la mas utilizada en el ecosistema Spring. Su clase central es `ObjectMapper`.

### Serializacion basica

```java
import com.fasterxml.jackson.databind.ObjectMapper;

public class JacksonBasico {
    record Usuario(Long id, String nombre, String email) {}

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        Usuario u = new Usuario(1L, "Ana", "ana@example.com");
        String json = mapper.writeValueAsString(u);
        System.out.println(json);

        Usuario usuario = mapper.readValue(json, Usuario.class);
        System.out.println(usuario.nombre());
    }
}
```

### Anotaciones Jackson

Jackson permite controlar la serializacion mediante anotaciones en las clases del modelo:

```java
import com.fasterxml.jackson.annotation.*;
import java.time.LocalDate;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Usuario {
    private Long id;

    @JsonProperty("nombre_completo")
    private String nombre;

    @JsonFormat(shape = JsonFormat.Shape.STRING,
            pattern = "dd/MM/yyyy")
    private LocalDate fechaNacimiento;

    @JsonIgnore
    private String password;

    // getters y setters
}
```

- `@JsonInclude(NON_NULL)`: omite campos con valor null en la salida.
- `@JsonProperty`: cambia el nombre del campo en el JSON.
- `@JsonFormat`: controla el formato de fechas y numeros.
- `@JsonIgnore`: excluye un campo de la serializacion/deserializacion.

### Custom serializers y deserializers

Para casos complejos se pueden escribir serializadores y deserializadores personalizados:

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.*;
import java.io.IOException;

class UsuarioSerializer extends JsonSerializer<Usuario> {
    @Override
    public void serialize(Usuario u, JsonGenerator gen,
            SerializerProvider prov) throws IOException {
        gen.writeStartObject();
        gen.writeNumberField("id", u.getId());
        gen.writeStringField("nombre", u.getNombre().toUpperCase());
        gen.writeEndObject();
    }
}

// Registrar en ObjectMapper
SimpleModule modulo = new SimpleModule();
modulo.addSerializer(Usuario.class, new UsuarioSerializer());
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(modulo);
```

### Configuracion global del ObjectMapper

```java
ObjectMapper mapper = new ObjectMapper()
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
    .configure(SerializationFeature.INDENT_OUTPUT, true)
    .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
    .setDateFormat(new SimpleDateFormat("yyyy-MM-dd"));
```

`FAIL_ON_UNKNOWN_PROPERTIES` en `false` evita errores cuando el JSON contiene campos que no existen en la clase Java, algo comun al evolucionar APIs.

> Si se usa Spring Boot, Jackson ya esta configurado y auto-configurado; solo es necesario anadir las dependencias `spring-boot-starter-web` y, opcionalmente, `jackson-datatype-jsr310` para soporte completo de `java.time`.

## Comparativa Gson vs Jackson

| Caracteristica | Gson | Jackson |
|---|---|---|
| Tamaño | ~250 KB | ~1.5 MB |
| Rendimiento | Bueno | Excelente |
| Anotaciones | Limitadas | Muy completas |
| Spring Boot | No incluido | Integrado |
| Custom serializers | `JsonSerializer<T>` | `JsonSerializer<T>` |
| Soporte `record` | Nativo (Java 16+) | Nativo (Jackson 2.12+) |

```xml
<!-- Gson -->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.10.1</version>
</dependency>

<!-- Jackson (core + databind) -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.0</version>
</dependency>
```

> Para proyectos nuevos que usen Spring Boot, Jackson es la opcion natural. Para proyectos ligeros o microservicios sin Spring, Gson reduce significativamente el peso de las dependencias.
