# Spring Data Redis

Spring Data Redis proporciona una integración completa entre Spring y Redis, el almacén de estructuras de datos en memoria más utilizado como caché, cola de mensajes y store de sesiones. Redis ofrece operaciones atómicas sobre tipos como strings, hashes, lists, sets y sorted sets, con tiempos de respuesta submilisegundo. Spring Data Redis abstrae estas operaciones mediante `RedisTemplate` y el patrón de repositorios, permitiendo usar Redis tanto como almacén primario de datos como capa de caché.

A diferencia de las bases de datos relacionales o documentales, Redis opera completamente en memoria, lo que lo hace ideal para datos que requieren acceso ultrarrápido pero que no necesitan persistencia garantizada. La integración con Spring Data Redis maneja automáticamente la serialización/deserialización de objetos Java a formatos Redis compatibles.

## Configuracion

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
      database: 0
      timeout: 500ms
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2
          max-wait: 500ms
```

Lettuce es el cliente Redis por defecto en Spring Boot, basado en Netty y completamente thread-safe. Reemplaza a Jedis como cliente recomendado gracias a su soporte nativo para conexiones reactivas y cluster.

## RedisTemplate

`RedisTemplate` es la interfaz de bajo nivel para interactuar con Redis. Permite ejecutar cualquier operación Redis con control completo sobre la serialización y el pipeline.

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}
```

```java
@Service
public class CacheService {
    private final RedisTemplate<String, Object> redisTemplate;

    public CacheService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void guardarProducto(String clave, Producto producto) {
        redisTemplate.opsForValue().set(clave, producto, Duration.ofMinutes(30));
    }

    public Producto obtenerProducto(String clave) {
        return (Producto) redisTemplate.opsForValue().get(clave);
    }

    public void guardarProductoEnHash(String hashKey, String field, Producto producto) {
        redisTemplate.opsForHash().put(hashKey, field, producto);
    }

    public void agregarAEspera(String listaKey, String elemento) {
        redisTemplate.opsForList().leftPush(listaKey, elemento);
    }

    public String sacarDeEspera(String listaKey) {
        return (String) redisTemplate.opsForList().rightPop(listaKey);
    }

    public void agregarASesion(String setId, String elemento) {
        redisTemplate.opsForSet().add(setId, elemento);
    }

    public void eliminar(String clave) {
        redisTemplate.delete(clave);
    }

    public boolean existe(String clave) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(clave));
    }
}
```

> `RedisTemplate` requiere serializers para convertir objetos Java a bytes. `GenericJackson2JsonRedisSerializer` serializa a JSON usando Jackson, lo que permite inspectar los datos almacenados en Redis con herramientas externas.

## StringRedisTemplate

`StringRedisTemplate` es una convenience class que utiliza `StringRedisSerializer` para ambas claves y valores, lo que la convierte en la opción predilecta para operaciones simples con strings.

```java
@Service
public class RateLimiterService {
    private final StringRedisTemplate redisTemplate;

    public RateLimiterService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public boolean permitirPeticion(String clienteId, int maxPeticiones, Duration ventana) {
        String clave = "rate:" + clienteId;
        Long contador = redisTemplate.opsForValue().increment(clave);

        if (contador != null && contador == 1) {
            redisTemplate.expire(clave, ventana);
        }

        return contador != null && contador <= maxPeticiones;
    }
}
```

```java
@Service
public class SessionService {
    private final StringRedisTemplate redisTemplate;

    public SessionService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void guardarSesion(String sessionId, String datos) {
        redisTemplate.opsForValue().set("session:" + sessionId, datos, Duration.ofHours(1));
    }

    public String obtenerSesion(String sessionId) {
        return redisTemplate.opsForValue().get("session:" + sessionId);
    }

    public void renovarSesion(String sessionId) {
        redisTemplate.expire("session:" + sessionId, Duration.ofHours(1));
    }
}
```

## @RedisHash

`@RedisHash` permite mapear entidades Java a estructuras Redis de forma automática, similar a `@Document` en MongoDB o `@Entity` en JPA.

```java
@RedisHash("productos")
public class Producto {

    @Id
    private String id;

    @Indexed
    private String nombre;

    private BigDecimal precio;

    @Indexed
    private String categoria;

    @TimeToLive
    private Long ttl; // Tiempo de vida en segundos

    public Producto() {}

    public Producto(String nombre, BigDecimal precio, String categoria) {
        this.nombre = nombre;
        this.precio = precio;
        this.categoria = categoria;
    }

    // Getters y setters
    public String getId() { return id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public BigDecimal getPrecio() { return precio; }
    public void setPrecio(BigDecimal precio) { this.precio = precio; }
    public String getCategoria() { return categoria; }
    public void setCategoria(String categoria) { this.categoria = categoria; }
    public Long getTtl() { return ttl; }
    public void setTtl(Long ttl) { this.ttl = ttl; }
}
```

```java
public interface ProductoRedisRepository extends CrudRepository<Producto, String> {

    List<Producto> findByCategoria(String categoria);

    List<Producto> findByNombreContaining(String termino);
}
```

```java
@Service
public class ProductoRedisService {
    private final ProductoRedisRepository repository;

    public ProductoRedisService(ProductoRedisRepository repository) {
        this.repository = repository;
    }

    public Producto guardar(Producto producto) {
        producto.setTtl(3600L); // TTL de 1 hora
        return repository.save(producto);
    }

    public List<Producto> buscarPorCategoria(String categoria) {
        return repository.findByCategoria(categoria);
    }
}
```

## Caching con Redis

Spring Framework ofrece abstracciones de caché que se integran directamente con Redis mediante `RedisCacheManager`.

```java
@SpringBootApplication
@EnableCaching
public class Aplicacion {
    public static void main(String[] args) {
        SpringApplication.run(Aplicacion.class, args);
    }
}
```

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutos en milisegundos
      cache-null-values: false
      key-prefix: "mi_app:"
      use-key-prefix: true
```

```java
@Service
public class ProductoCacheService {
    private final ProductoRepository productoRepository;

    public ProductoCacheService(ProductoRepository productoRepository) {
        this.productoRepository = productoRepository;
    }

    @Cacheable(value = "productos", key = "#id")
    public Optional<Producto> buscarPorId(Long id) {
        // Solo se ejecuta si no esta en cache
        simulateSlowService();
        return productoRepository.findById(id);
    }

    @CachePut(value = "productos", key = "#producto.id")
    public Producto actualizar(Producto producto) {
        return productoRepository.save(producto);
    }

    @CacheEvict(value = "productos", key = "#id")
    public void eliminar(Long id) {
        productoRepository.deleteById(id);
    }

    @CacheEvict(value = "productos", allEntries = true)
    public void limpiarCache() {
        // Elimina todas las entradas de la cache
    }

    private void simulateSlowService() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

> `@Cacheable` solo funciona en invocaciones externas al mismo bean. Si un método `@Cacheable` es llamado por otro método dentro de la misma clase, la cache no se intercepta. Este comportamiento se debe a que Spring usa proxies para la inyección de caché.

Redis como capa de caché ofrece ventajas significativas sobre otras soluciones: es compartido entre múltiples instancias de la aplicación, soporta expiración automática de claves y permite operaciones atómicas que garantizan consistencia en entornos concurrentes.

