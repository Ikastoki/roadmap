# Request/Response

Spring MVC proporciona una API flexible para manejar peticiones y respuestas HTTP mediante `ResponseEntity<T>` y `HttpHeaders`. Estas clases permiten un control granular sobre el contenido, headers y estado de las respuestas HTTP, siendo fundamentales para el desarrollo de APIs REST que cumplen con las mejores prácticas de HTTP.

`ResponseEntity<T>` representa toda la respuesta HTTP, incluyendo el body, headers y código de estado. Esta clase es la forma más completa de retornar respuestas en Spring MVC, permitiendo configurar cada aspecto de la respuesta de manera programática. `ResponseEntity` es especialmente útil cuando se necesita configurar headers específicos o retornar diferentes estados HTTP según la lógica de negocio.

```java
@RestController
@RequestMapping("/api/productos")
public class ProductoController {
    
    @GetMapping("/{id}")
    public ResponseEntity<Producto> obtener(@PathVariable Long id) {
        Optional<Producto> producto = productoService.obtenerPorId(id);
        
        if (producto.isPresent()) {
            HttpHeaders headers = new HttpHeaders();
            headers.add("X-Cache-Control", "max-age=3600");
            headers.add("X-Request-Id", UUID.randomUUID().toString());
            
            return ResponseEntity.ok()
                .headers(headers)
                .body(producto.get());
        } else {
            return ResponseEntity.notFound().build();
        }
    }
    
    @PostMapping
    public ResponseEntity<Producto> crear(@RequestBody @Valid Producto producto) {
        Producto creado = productoService.crear(producto);
        
        URI ubicacion = URI.create("/api/productos/" + creado.getId());
        
        return ResponseEntity.created(ubicacion)
            .header("X-Resource-Id", creado.getId().toString())
            .body(creado);
    }
    
    @GetMapping("/buscar")
    public ResponseEntity<List<Producto>> buscar(
            @RequestParam String nombre,
            @RequestParam int pagina,
            @RequestParam int tamano) {
        
        Page<Producto> resultados = productoService.buscar(nombre, PageRequest.of(pagina, tamano));
        
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-Total-Count", String.valueOf(resultados.getTotalElements()));
        headers.add("X-Total-Pages", String.valueOf(resultados.getTotalPages()));
        
        return ResponseEntity.ok()
            .headers(headers)
            .body(resultados.getContent());
    }
}
```

`HttpHeaders` es una clase que encapsula los headers HTTP de la petición o respuesta. Proporciona métodos tipados para establecer y obtener headers comunes, así como soporte para headers personalizados. Los headers son fundamentales para controlar el caché, la compresión, el formato de contenido y la seguridad de las respuestas.

`HttpStatus` es un enum que contiene todos los códigos de estado HTTP estándar. Utilizar `HttpStatus` en lugar de códigos numéricos mejora la legibilidad del código y reduce errores. Los códigos más comunes incluyen `OK` (200), `CREATED` (201), `NO_CONTENT` (204), `BAD_REQUEST` (400), `NOT_FOUND` (404) y `INTERNAL_SERVER_ERROR` (500).

La negociación de contenido (content negotiation) permite servir el mismo recurso en múltiples formatos. Spring MVC determina el formato de respuesta basándose en el header `Accept` de la petición, parámetros de URL o configuración por defecto. Esta funcionalidad es esencial para APIs que deben soportar múltiples clientes con diferentes requisitos de formato.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

Las anotaciones `consumes` y `produces` en `@RequestMapping` especifican los tipos de media que el controller puede manejar. `consumes` define qué tipos de media acepta el endpoint en el body de la petición, mientras que `produces` define qué tipos de media puede retornar. Esto permite crear endpoints especializados para formatos específicos.

```java
@RestController
@RequestMapping("/api/documentos")
public class DocumentoController {
    
    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    public Documento obtenerJson(@PathVariable Long id) {
        return documentoService.obtenerPorId(id);
    }
    
    @GetMapping(value = "/{id}/pdf", produces = MediaType.APPLICATION_PDF_VALUE)
    public byte[] obtenerPdf(@PathVariable Long id) {
        return documentoService.generarPdf(id);
    }
    
    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public Documento crearJson(@RequestBody Documento documento) {
        return documentoService.crear(documento);
    }
    
    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public Documento crearMultipart(@RequestParam MultipartFile archivo,
                                   @RequestParam String descripcion) {
        return documentoService.crear(archivo, descripcion);
    }
}
```

`@JsonView` permite controlar qué campos de un objeto se serializan a JSON según el contexto. Esta funcionalidad es útil cuando diferentes endpoints necesitan exponer diferentes subconjuntos de datos del mismo objeto. Se definen interfaces de "vista" que agrupan los campos, y se selecciona la vista apropiada en cada endpoint.

```java
public class Views {
    public static class Basico {}
    public static class Detallado extends Basico {}
}

public class Producto {
    @JsonView(Views.Basico.class)
    private Long id;
    
    @JsonView(Views.Basico.class)
    private String nombre;
    
    @JsonView(Views.Basico.class)
    private BigDecimal precio;
    
    @JsonView(Views.Detallado.class)
    private String descripcion;
    
    @JsonView(Views.Detallado.class)
    private List<Review> reviews;
}

@RestController
@RequestMapping("/api/productos")
public class ProductoController {
    
    @GetMapping
    @JsonView(Views.Basico.class)
    public List<Producto> listar() {
        return productoService.listar();
    }
    
    @GetMapping("/{id}")
    @JsonView(Views.Detallado.class)
    public Producto obtener(@PathVariable Long id) {
        return productoService.obtenerPorId(id);
    }
}
```

Los constructores estáticos de `ResponseEntity` simplifican la creación de respuestas comunes. `ResponseEntity.ok()` crea una respuesta 200, `ResponseEntity.notFound()` crea una 404, `ResponseEntity.badRequest()` crea una 400, y `ResponseEntity.status()` permite especificar cualquier código de estado. Estos constructores son más concisos que construir `ResponseEntity` manualmente.

```java
@GetMapping("/{id}")
public ResponseEntity<?> obtener(@PathVariable Long id) {
    try {
        Producto producto = productoService.obtenerPorId(id);
        return ResponseEntity.ok(producto);
    } catch (ResourceNotFoundException e) {
        return ResponseEntity.notFound()
            .header("X-Error-Message", e.getMessage())
            .build();
    } catch (Exception e) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(Map.of("error", e.getMessage()));
    }
}
```

> El uso de `ResponseEntity` en lugar de objetos simples proporciona control total sobre la respuesta HTTP. Esto es especialmente importante en APIs REST donde los headers personalizados, los códigos de estado específicos y la negociación de contenido son parte integral del diseño de la interfaz.

