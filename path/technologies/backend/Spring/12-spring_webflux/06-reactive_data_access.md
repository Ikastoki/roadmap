# Reactive Data Access

El acceso a datos en un contexto reactivo requiere que las operaciones de persistencia sean no-bloqueantes y devuelvan `Mono` o `Flux` en lugar de tipos sincronos. Spring Data ofrece modulos especializados que implementan esta filosofia sobre diferentes tecnologias de almacenamiento, permitiendo integrar bases de datos reactivas en el pipeline de WebFlux sin romper la cadena de ejecucion no-bloqueante.

## ReactiveCrudRepository

Spring Data Reactive extiende el patron `CrudRepository` con una variante reactiva. `ReactiveCrudRepository` define las mismas operaciones CRUD basicas pero retorna tipos reactivos: `Mono<T>` para operaciones que devuelven un unico elemento, y `Flux<T>` para colecciones.

```java
public interface UsuarioRepository extends ReactiveCrudRepository<Usuario, Long> {

    Flux<Usuario> findByActivoTrue();

    Mono<Usuario> findByEmail(String email);

    @Query("SELECT u FROM Usuario u WHERE u.fechaCreacion > :desde")
    Flux<Usuario> buscarCreadosDespues(@Param("desde") LocalDateTime desde);

    Flux<Usuario> findByDepartamentoNombre(String nombreDepartamento);
}
```

Las consultas derivadas del nombre de metodo se resuelven automaticamente igual que en Spring Data convencional, pero los retornos son siempre reactivos. `Mono<Usuario>` indica que se espera cero o un resultado, mientras que `Flux<Usuario>` indica una coleccion potencialmente vacia de elementos.

`ReactiveSortingRepository` extiende `ReactiveCrudRepository` con soporte para ordenamiento reactivo:

```java
public interface ProductoRepository extends ReactiveSortingRepository<Producto, Long> {

    Flux<Producto> findByCategoria(String categoria, Sort sort);
}
```

> `ReactiveCrudRepository` no soporta paginacion reactiva tradicional (`Pageable`) porque el concepto de paginacion por saltos (offset) es inherentemente bloqueante. En su lugar, se utiliza `Window` o `Buffer` sobre `Flux` para particionar resultados.

## R2DBC

**R2DBC** (Reactive Relational Database Connectivity) es la especificacion que define un API reactiva para bases de datos relacionales. Al igual que JDBC define un contrato para acceso bloqueante, R2DBC define un contrato para acceso no-bloqueante con backpressure nativa. Spring Data R2DBC integra esta especificacion con Spring Data.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
```

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mibd
    username: usuario
    password: secreto
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
```

El cliente R2DBC gestiona internamente un pool de conexiones no-bloqueantes. Cada operacion de consulta se ejecuta sin bloquear el hilo del reactor, y los resultados se emiten como elementos en un `Flux` conforme van disponibles desde la red.

```java
@Repository
public class PedidoRepositoryImpl implements PedidoRepositoryCustom {

    private final DatabaseClient databaseClient;

    public PedidoRepositoryImpl(DatabaseClient databaseClient) {
        this.databaseClient = databaseClient;
    }

    public Flux<PedidoResumen> buscarPedidosRecientes() {
        return databaseClient.sql(
                "SELECT p.id, p.total, c.nombre FROM pedido p " +
                "JOIN cliente c ON p.cliente_id = c.id " +
                "WHERE p.fecha > :fecha")
            .bind("fecha", LocalDateTime.now().minusDays(7))
            .map((row, metadata) -> new PedidoResumen(
                row.get("id", Long.class),
                row.get("total", BigDecimal.class),
                row.get("nombre", String.class)))
            .all();
    }
}
```

`DatabaseClient` es la herramienta de bajo nivel para ejecutar consultas SQL de forma reactiva, similar a `JdbcTemplate` pero no-bloqueante. Permite construir consultas de forma programatica y mapear resultados manualmente.

## Reactive MongoDB

MongoDB tiene soporte reactivo nativo a traves del driver `mongodb-driver-reactivestreams`. Spring Data MongoDB Reactive integra este driver con el ecosistema Spring, proporcionando `ReactiveMongoRepository` y `ReactiveMongoTemplate`.

```java
public interface EventoRepository extends ReactiveMongoRepository<Evento, String> {

    Flux<Evento> findByTipoAndTimestampAfter(TipoEvento tipo, Instant desde);

    @Query("{ 'metadata.usuarioId': ?0 }")
    Flux<Evento> eventosDeUsuario(String usuarioId);
}
```

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://usuario:secreto@localhost:27017/mibd?authSource=admin
      auto-index-creation: true
```

`ReactiveMongoTemplate` ofrece operaciones de bajo nivel con control total sobre la consulta:

```java
@Service
public class EventoService {

    private final ReactiveMongoTemplate mongoTemplate;

    public EventoService(ReactiveMongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    public Flux<Evento> buscarPorRango(Instant inicio, Instant fin) {
        Query query = new Query(Criteria
            .where("timestamp").gte(inicio).lte(fin))
            .with(Sort.by(Sort.Direction.DESC, "timestamp"))
            .limit(100);

        return mongoTemplate.find(query, Evento.class);
    }

    public Mono<Long> contarEventosActivos() {
        Query query = new Query(Criteria.where("activo").is(true));
        return mongoTemplate.count(query, Evento.class);
    }
}
```

## Reactive Redis

Redis es naturalmente no-bloqueante y su driver reactivo (`lettuce`) es el predeterminado en Spring Boot. `ReactiveRedisTemplate` y `ReactiveRedisConnectionFactory` proporcionan acceso reactivo completo a Redis.

```java
@Configuration
public class RedisConfig {

    @Bean
    public ReactiveRedisConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory("localhost", 6379);
    }

    @Bean
    public ReactiveRedisTemplate<String, Object> reactiveRedisTemplate(
            ReactiveRedisConnectionFactory factory) {
        return new ReactiveRedisTemplate<>(factory,
            RedisSerializationContext.string());
    }
}
```

```java
@Service
public class CacheService {

    private final ReactiveRedisTemplate<String, Object> redisTemplate;
    private static final Duration TTL = Duration.ofMinutes(30);

    public CacheService(ReactiveRedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public Mono<Boolean> guardar(String clave, Object valor) {
        return redisTemplate.opsForValue()
            .set(clave, valor, TTL);
    }

    public <T> Mono<T> obtener(String clave, Class<T> tipo) {
        return redisTemplate.opsForValue()
            .get(clave)
            .cast(tipo);
    }

    public Flux<String> claves(String patron) {
        return redisTemplate.keys(patron);
    }
}
```

La combinacion de R2DBC, Reactive MongoDB y Reactive Redis permite construir una capa de datos completamente reactiva que mantiene la no-bloqueacion desde el endpoint HTTP hasta la persistencia, eliminando los puntos de bloqueo que comprometerian el rendimiento de una aplicacion WebFlux.

