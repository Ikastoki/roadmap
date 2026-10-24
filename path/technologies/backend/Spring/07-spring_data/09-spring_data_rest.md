# Spring Data REST

Spring Data REST expone automáticamente los repositorios de Spring Data como endpoints RESTful, generando una API completa sin escribir un solo controller. Detecta las interfaces de repositorio, construye HATEOAS (Hypermedia as the Engine of Application State) y proporciona navegación entre recursos mediante enlaces hypermedia. Esto acelera significativamente el desarrollo de APIs CRUD pero requiere comprender sus convenciones y limitaciones para evitar sorprendentes en producción.

El objetivo principal de Spring Data REST es eliminar el código repetitivo de controllers que únicamente delegan en repositorios. Cuando la lógica de negocio es simple, esta aproximación elimina una capa completa de código. Sin embargo, cuando se necesita validación personalizada, transformaciones de datos o lógica transaccional compleja, es necesario recurrir a controllers personalizados.

## Configuracion basica

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

```java
@Entity
public class Producto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Size(min = 2, max = 100)
    private String nombre;

    @Column(precision = 10, scale = 2)
    private BigDecimal precio;

    @ManyToOne
    @JoinColumn(name = "categoria_id")
    private Categoria categoria;

    private boolean activo;

    public Producto() {}

    public Producto(String nombre, BigDecimal precio) {
        this.nombre = nombre;
        this.precio = precio;
        this.activo = true;
    }

    // Getters y setters
    public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public BigDecimal getPrecio() { return precio; }
    public void setPrecio(BigDecimal precio) { this.precio = precio; }
    public Categoria getCategoria() { return categoria; }
    public void setCategoria(Categoria categoria) { this.categoria = categoria; }
    public boolean isActivo() { return activo; }
    public void setActivo(boolean activo) { this.activo = activo; }
}
```

```java
@Entity
public class Categoria {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;

    @OneToMany(mappedBy = "categoria")
    private List<Producto> productos;

    public Categoria() {}

    public Categoria(String nombre) {
        this.nombre = nombre;
    }

    // Getters y setters
    public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public List<Producto> getProductos() { return productos; }
    public void setProductos(List<Producto> productos) { this.productos = productos; }
}
```

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {
}

public interface CategoriaRepository extends JpaRepository<Categoria, Long> {
}
```

Con esta configuración mínima, Spring Data REST genera automáticamente los siguientes endpoints:

| Metodo | Endpoint               | Descripcion                     |
|--------|------------------------|---------------------------------|
| GET    | `/api/productos`       | Listar todos (paginado)         |
| GET    | `/api/productos/{id}`  | Obtener por ID                  |
| POST   | `/api/productos`       | Crear nuevo                     |
| PUT    | `/api/productos/{id}`  | Actualizar completo             |
| PATCH  | `/api/productos/{id}`  | Actualizar parcial              |
| DELETE | `/api/productos/{id}`  | Eliminar                        |
| GET    | `/api/categorias`      | Listar categorias               |
| GET    | `/api/categorias/{id}` | Obtener categoria con enlaces   |

## Formato HAL

Spring Data REST retorna respuestas en formato HAL (Hypertext Application Language), un formato estándar para APIs hypermedia que incluye enlaces para navegar entre recursos.

```json
{
  "_embedded": {
    "productos": [
      {
        "nombre": "Laptop Pro",
        "precio": 1299.99,
        "activo": true,
        "_links": {
          "self": {
            "href": "http://localhost:8080/api/productos/1"
          },
          "producto": {
            "href": "http://localhost:8080/api/productos/1"
          },
          "categoria": {
            "href": "http://localhost:8080/api/productos/1/categoria"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8080/api/productos"
    },
    "profile": {
      "href": "http://localhost:8080/api/profile/productos"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 150,
    "totalPages": 8,
    "number": 0
  }
}
```

## Paginacion y ordenacion

Los repositorios exponen paginación y ordenación de forma automática mediante parámetros de consulta.

```bash
# Obtener pagina 2 con 10 elementos
GET /api/productos?page=1&size=10

# Ordenar por precio descendente
GET /api/productos?sort=precio,desc

# Combinar paginacion y ordenacion
GET /api/productos?page=0&size=25&sort=nombre,asc
```

```yaml
# application.yml - configuracion por defecto
spring:
  data:
    rest:
      default-page-size: 20
      max-page-size: 100
      base-path: /api
```

## @RepositoryRestController

`@RepositoryRestController` permite personalizar el comportamiento de los endpoints generados automáticamente sin perder la integración con Spring Data REST.

```java
@RepositoryRestController
@RequestMapping("/api/productos")
public class ProductoController {

    private final ProductoRepository productoRepository;
    private final ProductoService productoService;

    public ProductoController(ProductoRepository productoRepository,
                             ProductoService productoService) {
        this.productoRepository = productoRepository;
        this.productoService = productoService;
    }

    // Sobreescribir el endpoint de creacion
    @PostMapping
    public ResponseEntity<?> crearProducto(@RequestBody @Valid Producto producto) {
        Producto creado = productoService.crearConValidacion(producto);
        EntityModel<Producto> modelo = EntityModel.of(creado);
        modelo.add(linkTo(methodOn(ProductoController.class).crearProducto(producto))
                .withSelfRel());
        return ResponseEntity.status(HttpStatus.CREATED).body(modelo);
    }

    // Endpoint adicional
    @GetMapping("/estadisticas")
    public ResponseEntity<?> obtenerEstadisticas() {
        Map<String, Object> stats = new HashMap<>();
        stats.put("total", productoRepository.count());
        stats.put("activos", productoRepository.countByActivo(true));
        return ResponseEntity.ok(stats);
    }
}
```

```java
// Servicio con logica de negocio
@Service
public class ProductoService {
    private final ProductoRepository productoRepository;

    public ProductoService(ProductoRepository productoRepository) {
        this.productoRepository = productoRepository;
    }

    @Transactional
    public Producto crearConValidacion(Producto producto) {
        if (producto.getPrecio().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("El precio debe ser mayor que cero");
        }
        if (productoRepository.existsByNombre(producto.getNombre())) {
            throw new IllegalArgumentException("Ya existe un producto con ese nombre");
        }
        return productoRepository.save(producto);
    }
}
```

> Spring Data REST busca anotaciones `@RepositoryRestController` en el paquete base de la aplicación y sus subpaquetes. Los controllers personalizados sobreescriben los endpoints generados automáticamente para las mismas rutas.

## Proyecciones y excerpt projections

Spring Data REST soporta proyecciones para controlar qué campos se exponen en las respuestas.

```java
// Proyeccion para listas
@Projection(name = "basico", types = {Producto.class})
public interface ProductoBasico {
    String getNombre();
    BigDecimal getPrecio();
}

// Proyeccion con relaciones
@Projection(name = "detallado", types = {Producto.class})
public interface ProductoDetallado {
    String getNombre();
    BigDecimal getPrecio();
    Categoria getCategoria();
}

// Excerpt projection: se aplica automaticamente a las colecciones
@ProjectedPayload
public interface ProductoExcerpt {
    String getNombre();
    BigDecimal getPrecio();
}
```

```java
public interface ProductoRepository extends PagingAndSortingRepository<Producto, Long> {
    // Las proyecciones se activan via query param: /api/productos?projection=basico
}
```

Spring Data REST automatiza la creación de APIs RESTful completas, pero su uso debe ser consciente de las limitaciones: la personalización se vuelve más compleja a medida que crecen los requisitos, y la exposición directa de entidades puede revelar información sensible si no se configuran proyecciones adecuadas.

