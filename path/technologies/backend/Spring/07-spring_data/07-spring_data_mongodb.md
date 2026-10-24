# Spring Data MongoDB

Spring Data MongoDB proporciona una integración profunda entre Spring y MongoDB, el base de datos NoSQL más popular del ecosistema de documentos. A diferencia de las bases de datos relacionales, MongoDB almacena datos en documentos BSON (JSON binario), lo que elimina la necesidad de esquemas rígidos y relaciones JOIN. Spring Data MongoDB adapta el patrón de repositorios de Spring a este modelo de datos, ofreciendo una experiencia de desarrollo consistente con Spring Data JPA.

MongoDB almacena documentos en colecciones (equivalentes a tablas), y cada documento tiene un identificador único `_id`. Spring Data MongoDB maneja este identificador de forma transparente, mapeándolo a un campo `@Id` en la entidad.

## Anotacion @Document

`@Document` marca una clase como una entidad MongoDB que se persiste en una colección específica. A diferencia de `@Entity` en JPA, `@Document` no requiere un esquema predefinido y permite campos opcionales sin definición previa.

```java
@Document(collection = "productos")
public class Producto {

    @Id
    private String id;

    @Field("nombre_producto")
    private String nombre;

    private BigDecimal precio;

    @Indexed
    private String categoria;

    private boolean activo;

    @CreatedDate
    private LocalDateTime fechaCreacion;

    @LastModifiedDate
    private LocalDateTime fechaModificacion;

    // Constructor por defecto requerido por MongoDB
    public Producto() {}

    public Producto(String nombre, BigDecimal precio, String categoria) {
        this.nombre = nombre;
        this.precio = precio;
        this.categoria = categoria;
        this.activo = true;
    }

    // Getters y setters
    public String getId() { return id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public BigDecimal getPrecio() { return precio; }
    public void setPrecio(BigDecimal precio) { this.precio = precio; }
    public String getCategoria() { return categoria; }
    public void setCategoria(String categoria) { this.categoria = categoria; }
    public boolean isActivo() { return activo; }
    public void setActivo(boolean activo) { this.activo = activo; }
}
```

```java
// Configuracion en application.yml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mi_base_datos
      # Alternativa con propiedades individuales
      # host: localhost
      # port: 27017
      # database: mi_base_datos
```

> MongoDB genera automáticamente el `_id` si no se especifica al insertar un documento. El tipo puede ser `String`, `ObjectId` o cualquier tipo serializable.

## MongoRepository

`MongoRepository` extiende `PagingAndSortingRepository` y `QueryByExampleExecutor`, proporcionando las mismas operaciones CRUD y paginación que `JpaRepository` pero optimizadas para MongoDB.

```java
public interface ProductoRepository extends MongoRepository<Producto, String> {

    List<Producto> findByCategoria(String categoria);

    List<Producto> findByPrecioLessThan(BigDecimal precio);

    @Query("{ 'categoria': ?0, 'activo': true }")
    List<Producto> buscarActivosPorCategoria(String categoria);

    @Query("{ 'precio': { $gte: ?0, $lte: ?1 } }")
    List<Producto> buscarPorRangoPrecio(BigDecimal min, BigDecimal max);

    @Query(value = "{ 'categoria': ?0 }", fields = "{ 'nombre': 1, 'precio': 1 }")
    List<Producto> proyectarPorCategoria(String categoria);

    long countByCategoria(String categoria);

    void deleteByActivoFalse();
}
```

```java
@Service
public class ProductoService {
    private final ProductoRepository productoRepository;

    public ProductoService(ProductoRepository productoRepository) {
        this.productoRepository = productoRepository;
    }

    public Producto crearProducto(Producto producto) {
        return productoRepository.save(producto);
    }

    public List<Producto> buscarPorCategoria(String categoria) {
        return productoRepository.findByCategoria(categoria);
    }

    public Page<Producto> productosPaginados(int pagina, int tamanio) {
        Pageable pageable = PageRequest.of(pagina, tamanio, Sort.by("precio").ascending());
        return productoRepository.findAll(pageable);
    }
}
```

## MongoTemplate

`MongoTemplate` es la interfaz de bajo nivel para operaciones MongoDB que requieren control directo sobre las consultas, actualizaciones condicionales o operaciones de colección que los repositorios no exponen.

```java
@Service
public class ProductoMongoService {
    private final MongoTemplate mongoTemplate;

    public ProductoMongoService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    // Busqueda con condiciones complejas
    public List<Producto> buscarProductosAvanzados(String categoria, BigDecimal precioMin,
                                                    BigDecimal precioMax) {
        Query query = new Query();
        query.addCriteria(Criteria.where("categoria").is(categoria)
                .and("precio").gte(precioMin).lte(precioMax)
                .and("activo").is(true));
        query.with(Sort.by(Sort.Direction.DESC, "precio"));
        query.limit(50);

        return mongoTemplate.find(query, Producto.class);
    }

    // Actualizacion parcial
    public void actualizarStock(String productoId, int nuevoStock) {
        Update update = new Update();
        update.set("stock", nuevoStock);
        update.set("fechaModificacion", LocalDateTime.now());

        mongoTemplate.updateFirst(
            Query.query(Criteria.where("id").is(productoId)),
            update,
            Producto.class
        );
    }

    // Upsert: insertar o actualizar
    public Producto upsert(Producto producto) {
        return mongoTemplate.save(producto);
    }

    // Contar documentos con condiciones
    public long contarActivos(String categoria) {
        Query query = Query.query(
            Criteria.where("categoria").is(categoria).and("activo").is(true)
        );
        return mongoTemplate.count(query, Producto.class);
    }
}
```

> `MongoTemplate` opera directamente sobre colecciones, sin pasar por el entity manager. Esto lo hace más rápido pero más propenso a errores si no se verifica la consistencia del esquema de los documentos.

## Consultas con @Query y JSON

Las consultas en Spring Data MongoDB se definen usando la sintaxis JSON de MongoDB, que es diferente a JPQL. Los operadores como `$gte`, `$in`, `$regex` y `$and` se utilizan directamente en la anotación.

```java
public interface ProductoRepository extends MongoRepository<Producto, String> {

    // Consulta con $regex
    @Query("{ 'nombre': { $regex: ?0, $options: 'i' } }")
    List<Producto> buscarPorNombreRegex(String patron);

    // Consulta con $in
    @Query("{ 'categoria': { $in: ?0 } }")
    List<Producto> buscarEnCategorias(List<String> categorias);

    // Consulta con $and
    @Query("{ $and: [ { 'precio': { $gte: ?0 } }, { 'activo': true } ] }")
    List<Producto> buscarCarosActivos(BigDecimal precioMin);

    // Proyeccion con campos especificos
    @Query(value = "{ 'categoria': ?0 }", fields = "{ 'nombre': 1, 'precio': 1, '_id': 0 }")
    List<Map<String, Object>> proyectarCampos(String categoria);

    // Consulta con sort nativo
    @Query(value = "{ 'activo': true }", sort = "{ 'precio': -1 }")
    List<Producto> activosOrdenadosPorPrecio();
}
```

## Change Streams

Los change streams permiten escuchar cambios en tiempo real en una colección o base de datos de MongoDB. Spring Data MongoDB soporta change streams mediante `MongoTemplate` o la API reactiva.

```java
@Service
public class ChangeStreamService {
    private final MongoTemplate mongoTemplate;

    public ChangeStreamService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    @PostConstruct
    public void iniciarEscucha() {
        ChangeStreamRequest<Producto> request = ChangeStreamRequest.builder()
                .collection("productos")
                .filter(Criteria.where("operationType").in("insert", "update", "delete"))
                .launchWith(Collections.emptyList())
                .addEventsListener((ChangeStreamEvent<Producto> event) -> {
                    System.out.println("Operacion detectada: " + event.getEventType());
                    System.out.println("Documento: " + event.getBody());
                })
                .build();

        mongoTemplate.watch(request, Producto.class);
    }
}
```

```java
// Change stream reactivo con MongoReactiveClient
@Configuration
public class ChangeStreamConfig {

    @Bean
    public SubscriptionChangeStreamProcessor changeStreamProcessor(
            ReactiveMongoTemplate reactiveMongoTemplate) {

        ChangeStreamOptions options = ChangeStreamOptions.builder()
                .filter(Aggregation.match(Criteria.where("operationType").is("insert")))
                .build();

        return reactiveMongoTemplate.changeStream("productos", options, Producto.class)
                .doOnNext(event -> {
                    System.out.println("Nuevo producto: " + event.getBody());
                })
                .subscribe();
    }
}
```

> Los change streams requieren una configuración de réplica set en MongoDB. En entornos de desarrollo con un solo nodo, se puede habilitar con `mongod --replSet rs0`.

Spring Data MongoDB adapta el patrón de repositorios de Spring al modelo de documentos, permitiendo transicionar entre bases de datos relacionales y no relacionales sin cambiar la arquitectura de la aplicación.

