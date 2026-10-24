# Package Structure

La estructura de paquetes de una aplicacion Spring Boot determina la mantenibilidad, escalabilidad y legibilidad del codigo a lo largo del tiempo. Una estructura mal organizada genera dependencias ciclicas, dificulta las pruebas y aumenta el acoplamiento entre componentes. La estructura recomendada sigue el patron de arquitectura por capas donde cada paquete tiene una responsabilidad unica y definida.

## Estructura base

```
com.ejemplo.miaplicacion/
├── config/
│   ├── DatabaseConfig.java
│   ├── SecurityConfig.java
│   ├── AsyncConfig.java
│   └── CacheConfig.java
├── controller/
│   ├── UsuarioController.java
│   ├── PedidoController.java
│   └── ProductoController.java
├── service/
│   ├── UsuarioService.java
│   ├── PedidoService.java
│   └── ProductoService.java
├── service/impl/
│   ├── UsuarioServiceImpl.java
│   ├── PedidoServiceImpl.java
│   └── ProductoServiceImpl.java
├── repository/
│   ├── UsuarioRepository.java
│   ├── PedidoRepository.java
│   └── ProductoRepository.java
├── model/
│   ├── entity/
│   │   ├── Usuario.java
│   │   ├── Pedido.java
│   │   └── Producto.java
│   └── domain/
│       ├── UsuarioId.java
│       └── PedidoEstado.java
├── dto/
│   ├── request/
│   │   ├── CrearUsuarioDTO.java
│   │   └── ActualizarPedidoDTO.java
│   └── response/
│       ├── UsuarioDTO.java
│       ├── PedidoDTO.java
│       └── ErrorResponseDTO.java
├── exception/
│   ├── ResourceNotFoundException.java
│   ├── BusinessException.java
│   ├── ValidationException.java
│   └── GlobalExceptionHandler.java
├── util/
│   ├── DateUtils.java
│   └── StringUtils.java
└── MiAplicacion.java
```

## Capa de configuracion (config)

El paquete `config` contiene clases de configuracion de Spring que definen beans, comportamientos y aspectos transversales. Estas clases utilizan `@Configuration` y definen `@Bean` methods que configuran la infraestructura de la aplicacion: datasources, pools de conexiones, clientes HTTP, seguridad, cache y other aspectos tecnicos.

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

## Capa de controladores (controller)

Los controladores manejan las peticiones HTTP y delegan la logica de negocio a los servicios. Cada controlador se encarga de un recurso o dominio especifico y se encarga de la validacion de entrada, la transformacion de DTOs y la construccion de respuestas HTTP.

```java
@RestController
@RequestMapping("/api/v1/usuarios")
@Tag(name = "Usuarios", description = "Gestion de usuarios")
public class UsuarioController {

    private final UsuarioService usuarioService;

    public UsuarioController(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UsuarioDTO> obtener(@PathVariable Long id) {
        return ResponseEntity.ok(usuarioService.obtenerPorId(id));
    }

    @PostMapping
    public ResponseEntity<UsuarioDTO> crear(
            @RequestBody @Valid CrearUsuarioDTO dto) {
        UsuarioDTO creado = usuarioService.crear(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(creado);
    }
}
```

## Capa de servicios (service/impl)

La capa de servicios contiene la logica de negocio. La interfaz `service` define el contrato, mientras que `service/impl` contiene la implementacion. Esta separacion permite inyectar implementaciones alternativas en pruebas y mantener la interfaz limpia de dependencias de infraestructura.

```java
public interface UsuarioService {
    UsuarioDTO obtenerPorId(Long id);
    List<UsuarioDTO> listar(int pagina, int tamano);
    UsuarioDTO crear(CrearUsuarioDTO dto);
    UsuarioDTO actualizar(Long id, ActualizarUsuarioDTO dto);
    void eliminar(Long id);
}
```

## Capa de repositorios (repository)

Los repositorios encapsulan el acceso a datos. Utilizan Spring Data para definir interfaces que se implementan automaticamente, o clases de implementacion personalizada para consultas complejas.

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByEmail(String email);
    boolean existsByEmail(String email);

    @Query("SELECT u FROM Usuario u WHERE u.activo = true " +
           "AND u.fechaCreacion > :desde")
    List<Usuario> buscarActivosRecientes(@Param("desde") LocalDateTime desde);
}
```

## Model y DTO

`model` contiene las entidades de dominio (entidades JPA) y tipos de valor. `dto` contiene los objetos de transferencia de datos separados en `request` (entrada) y `response` (salida). Esta separacion evita exponer la estructura interna de la base de datos en la API y permite evolucionar el modelo de persistencia sin afectar a los clientes.

## Capa de excepciones (exception)

`exception` contiene las excepciones personalizadas de dominio y el handler global que transforma excepciones en respuestas HTTP estandarizadas. Centralizar el manejo de excepciones garantiza consistencia en los mensajes de error y facilita el logging.

