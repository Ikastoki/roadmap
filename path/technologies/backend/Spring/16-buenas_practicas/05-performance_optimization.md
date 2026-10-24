# Performance Optimization

La optimizacion de rendimiento en aplicaciones Spring Boot abarca multiples dimensiones: desde la gestion eficiente de conexiones de base de datos hasta la configuracion de la JVM, pasando por el uso correcto de cache y procesamiento asincrono. Cada optimizacion debe medirse y validarse con benchmarks antes de implementarse, porque una optimizacion prematura puede introducir complejidad sin beneficios medibles.

## Connection pooling con HikariCP

HikariCP es el pool de conexiones predeterminado en Spring Boot y uno de los mas eficientes del ecosistema Java. La configuracion correcta del pool impacta directamente en la latencia de las operaciones de base de datos y en la capacidad de la aplicacion para manejar concurrencia.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 600000
      connection-timeout: 10000
      leak-detection-threshold: 60000
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        rewriteBatchedStatements: true
```

> El `maximum-pool-size` debe ajustarse segun la capacidad de la base de datos. Un pool demasiado grande puede saturar las conexiones del servidor de BD, mientras que uno demasiado pequeno limita la concurrencia de la aplicacion.

## Optimizacion de consultas

Las consultas ineficientes son la causa mas comun de problemas de rendimiento en aplicaciones con base de datos relacional. Las practicas recomendadas incluyen evitar `SELECT *`, usar indices adecuados, paginar resultados y evitar el problema N+1.

```java
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    // MAL: N+1 query - una consulta por cada pedido para obtener items
    @Query("SELECT p FROM Pedido p WHERE p.estado = :estado")
    List<Pedido> findByEstado(@Param("estado") PedidoEstado estado);

    // BIEN: fetch join - una unica consulta con joins
    @Query("SELECT DISTINCT p FROM Pedido p " +
           "JOIN FETCH p.items " +
           "JOIN FETCH p.cliente " +
           "WHERE p.estado = :estado")
    List<Pedido> findByEstadoConItems(@Param("estado") PedidoEstado estado);

    // BIEN: paginacion para colecciones grandes
    @Query("SELECT p FROM Pedido p WHERE p.fechaCreacion > :desde")
    Page<Pedido> buscarRecientes(@Param("desde") LocalDateTime desde, Pageable pageable);

    // BIEN: proyeccion para reducir transferencia de datos
    @Query("SELECT new com.ejemplo.dto.PedidoResumenDTO(p.id, p.total, p.fechaCreacion) " +
           "FROM Pedido p WHERE p.estado = :estado")
    List<PedidoResumenDTO> resumenPorEstado(@Param("estado") PedidoEstado estado);
}
```

## Cache con Spring Cache

La cache evita consultas repetitivas a fuentes de datos lentas. Spring Cache abstraction permite integrar multiples proveedores (Redis, Ehcache, Caffeine) con una API unica.

```java
@Service
@Slf4j
public class ProductoService {

    private final ProductoRepository repository;

    public ProductoService(ProductoRepository repository) {
        this.repository = repository;
    }

    @Cacheable(value = "productos", key = "#id", unless = "#result == null")
    public ProductoDTO obtenerPorId(Long id) {
        log.debug("Cache miss para producto {}", id);
        return repository.findById(id)
            .map(this::toDTO)
            .orElse(null);
    }

    @CachePut(value = "productos", key = "#resultado.id")
    public ProductoDTO actualizar(ProductoDTO dto) {
        Producto entidad = repository.save(mapper.toEntity(dto));
        return toDTO(entidad);
    }

    @CacheEvict(value = "productos", key = "#id")
    public void eliminar(Long id) {
        repository.deleteById(id);
    }

    @CacheEvict(value = "productos", allEntries = true)
    public void limpiarCache() {
        log.info("Cache de productos limpiada");
    }
}
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CaffeineCacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(30))
            .recordStats());
        return manager;
    }
}
```

## Procesamiento asincrono con @Async

`@Async` permite ejecutar metodos en un pool de hilos separado sin bloquear el hilo de la peticion. Es ideal para operaciones que no necesitan esperar la respuesta: envio de emails, generacion de reportes o sincronizacion de datos.

```java
@Service
@Slf4j
public class NotificacionService {

    @Async
    public CompletableFuture<Boolean> enviarEmail(EmailDTO email) {
        try {
            // Envio asincrono del email
            mailSender.send(mimeMessage -> {
                mimeMessage.setTo(email.getDestinatario());
                mimeMessage.setSubject(email.getAsunto());
                mimeMessage.setText(email.getCuerpo(), true, "HTML");
            });
            log.info("Email enviado a {}", email.getDestinatario());
            return CompletableFuture.completedFuture(true);
        } catch (Exception e) {
            log.error("Error enviando email a {}: {}",
                      email.getDestinatario(), e.getMessage());
            return CompletableFuture.completedFuture(false);
        }
    }

    @Async
    @Retryable(retryFor = Exception.class, maxAttempts = 3,
               backoff = @Backoff(delay = 1000))
    public void sincronizarDatos(EntidadSync datos) {
        // Sincronizacion pesada con reintento automatico
    }
}
```

## Lazy initialization

La inicializacion perezosa de beans retrasa la creacion de componentes que no se usan inmediatamente, reduciendo el tiempo de arranque de la aplicacion.

```yaml
spring:
  main:
    lazy-initialization: true
```

```java
@Lazy
@Component
public class ReporteGenerator {

    @PostConstruct
    public void init() {
        // Solo se ejecuta cuando el bean se necesita por primera vez
        log.info("Inicializando generador de reportes");
    }
}
```

> La lazy initialization reduce el tiempo de arranque pero puede enmascarar errores de configuracion que solo se descubren cuando el bean se usa por primera vez. En produccion, se recomienda ejecutar un health check completo tras el arranque para validar que todos los beans se inicializan correctamente.

## JVM tuning

La configuracion de la JVM impacta directamente en el rendimiento y el consumo de memoria de la aplicacion. Los parametros mas relevantes para aplicaciones en contenedores son:

```bash
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+UseContainerSupport \
     -XX:MaxRAMPercentage=75.0 \
     -XX:InitialRAMPercentage=50.0 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     -Djava.security.egd=file:/dev/./urandom \
     -jar mi-aplicacion.jar
```

`-XX:+UseContainerSupport` es fundamental en entornos Docker/Kubernetes porque permite a la JVM detectar automaticamente los limites de memoria y CPU del contenedor. Sin esta opcion, la JVM puede intentar reservar mas memoria de la disponible, provocando un `OutOfMemoryError` y la terminacion del contenedor por parte de Kubernetes.

