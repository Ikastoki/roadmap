# Spring Data JPA

Spring Data JPA es el módulo del proyecto Spring Data que proporciona soporte completo para persistencia JPA (Java Persistence API). Se integra transparentemente con proveedores como Hibernate, EclipseLink o OpenJPA, ofreciendo una capa de abstracción que simplifica la configuración, el mapeo de entidades y la ejecución de consultas. Su objetivo es reducir significativamente la cantidad de código necesario para trabajar con bases de datos relacionales en aplicaciones Spring.

La integración con JPA en Spring no consiste únicamente en envolver las interfaces de JPA en beans del contenedor. Spring Data JPA analiza las interfaces de repositorio, genera implementaciones concretas en tiempo de ejecución y gestiona el ciclo de vida de las entidades dentro del contexto de persistencia de forma automática.

## Auto-configuracion

Spring Boot detecta automáticamente la presencia de `spring-boot-starter-data-jpa` y configura el `EntityManagerFactory`, el `TransactionManager` y los repositorios de forma convencional. Para que la auto-configuración funcione, se necesita al menos un datasource y un proveedor JPA válido.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mi_base
    username: usuario
    password: clave
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

> El `ddl-auto` controla cómo Hibernate gestiona el esquema de la base de datos. En producción se recomienda `validate` o `none`, mientras que en desarrollo `update` o `create-drop` son opciones prácticas.

## Hibernate como proveedor por defecto

Cuando se incluye `spring-boot-starter-data-jpa`, Spring Boot incluye Hibernate como proveedor JPA por defecto. Hibernate implementa la especificación JPA y añade funcionalidades propias que se acceden mediante su API nativa cuando es necesario.

```java
// Entidad mapeada con Hibernate
@Entity
@Table(name = "productos", schema = "inventario")
public class Producto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(precision = 10, scale = 2)
    private BigDecimal precio;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private EstadoProducto estado;

    @CreationTimestamp
    @Column(updatable = false)
    private LocalDateTime fechaCreacion;

    @UpdateTimestamp
    private LocalDateTime fechaModificacion;

    public Producto() {}

    public Producto(String nombre, BigDecimal precio) {
        this.nombre = nombre;
        this.precio = precio;
        this.estado = EstadoProducto.ACTIVO;
    }

    // Getters y setters
    public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public BigDecimal getPrecio() { return precio; }
    public void setPrecio(BigDecimal precio) { this.precio = precio; }
    public EstadoProducto getEstado() { return estado; }
    public void setEstado(EstadoProducto estado) { this.estado = estado; }
}
```

```java
public enum EstadoProducto {
    ACTIVO,
    DESCONTINUADO,
    AGOTADO
}
```

## Generacion DDL automatica

El `ddl-auto` es el parámetro que controla la generación automática del esquema de base de datos. Cada proveedor JPA implementa esta funcionalidad de forma diferente, pero Spring Boot unifica la configuración mediante este único parámetro.

| Valor       | Comportamiento                                                |
|-------------|---------------------------------------------------------------|
| `none`      | No hace nada. El esquema debe gestionarse externamente        |
| `validate`  | Valida que el esquema coincida con las entidades               |
| `update`    | Actualiza el esquema agregando columnas/tablas faltantes       |
| `create`    | Crea el esquema al iniciar, eliminando datos existentes       |
| `create-drop` | Crea al iniciar y elimina al cerrar la JVM                  |

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # Recomendado en produccion
```

> En entornos de producción, la generación automática del esquema puede ser peligrosa. Se recomienda herramientas externas como Flyway o Liquibase para gestionar migraciones de esquema de forma controlada y versionada.

## Inicializacion de base de datos

Spring Boot puede ejecutar scripts SQL o JPA al iniciar la aplicación para sembrar datos iniciales o verificar el esquema.

```yaml
spring:
  sql:
    init:
      mode: embedded        # embedded | always | never
      schema-locations: classpath:schema.sql
      data-locations: classpath:data.sql
      platform: postgresql
```

```sql
-- schema.sql
CREATE TABLE IF NOT EXISTS usuarios (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    activo BOOLEAN DEFAULT true,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- data.sql
INSERT INTO usuarios (username, email) VALUES ('admin', 'admin@ejemplo.com');
INSERT INTO usuarios (username, email) VALUES ('user1', 'user1@ejemplo.com');
```

```java
// Configuracion alternativa con CommandLineRunner
@SpringBootApplication
public class Aplicacion {

    public static void main(String[] args) {
        SpringApplication.run(Aplicacion.class, args);
    }

    @Bean
    CommandLineRunner init(UsuarioRepository usuarioRepository) {
        return args -> {
            if (usuarioRepository.count() == 0) {
                usuarioRepository.save(new Usuario("admin", "admin@ejemplo.com"));
                usuarioRepository.save(new Usuario("user1", "user1@ejemplo.com"));
            }
        };
    }
}
```

## Fuentes de datos y transacciones

Spring Data JPA gestiona las transacciones mediante `@Transactional`, que se puede aplicar a nivel de servicio o de método. La transacción se inicia antes de ejecutar el método y se confirma o revierte automáticamente según el resultado.

```java
@Service
@Transactional
public class ProductoService {

    private final ProductoRepository productoRepository;
    private final InventarioRepository inventarioRepository;

    public ProductoService(ProductoRepository productoRepository,
                          InventarioRepository inventarioRepository) {
        this.productoRepository = productoRepository;
        this.inventarioRepository = inventarioRepository;
    }

    // Transaccion de lectura
    @Transactional(readOnly = true)
    public Optional<Producto> buscarPorId(Long id) {
        return productoRepository.findById(id);
    }

    // Transaccion de escritura
    public Producto crearProducto(Producto producto) {
        return productoRepository.save(producto);
    }

    // Transaccion con rollback personalizado
    @Transactional(rollbackFor = StockInsuficienteException.class)
    public void transferirStock(Long origenId, Long destinoId, int cantidad)
            throws StockInsuficienteException {
        Producto origen = productoRepository.findById(origenId)
                .orElseThrow(() -> new IllegalArgumentException("Producto origen no encontrado"));
        Producto destino = productoRepository.findById(destinoId)
                .orElseThrow(() -> new IllegalArgumentException("Producto destino no encontrado"));

        if (origen.getStock() < cantidad) {
            throw new StockInsuficienteException("Stock insuficiente en producto origen");
        }

        origen.setStock(origen.getStock() - cantidad);
        destino.setStock(destino.getStock() + cantidad);

        productoRepository.save(origen);
        productoRepository.save(destino);
    }
}
```

> `@Transactional(readOnly = true)` optimiza la conexión de base de datos al indicar al proveedor JPA que la transacción solo realizará operaciones de lectura. Esto puede habilitar optimizaciones específicas del proveedor, como evitar la escritura del log de transacciones.

La autoconfiguración de Spring Data JPA oculta complejidad significativa: gestión del `EntityManagerFactory`, pooling de conexiones, traducción de excepciones del proveedor a excepciones de Spring y demarcación transaccional. Esto permite al desarrollador centrarse en la lógica de negocio sin preocuparse por los detalles de infraestructura de persistencia.

