# Configuration Management

La gestion de configuracion en aplicaciones Spring Boot determina la flexibilidad, seguridad y mantenibilidad del sistema en diferentes entornos. Una buena gestion de configuracion separa los valores que cambian entre entornos (credenciales, URLs, tamanos de pool) del codigo que permanece constante, y utiliza tipos fuertemente tipados para detectar errores de configuracion durante el arranque en lugar de en tiempo de ejecucion.

## Configuracion externalizada

La configuracion externalizada permite ejecutar la misma imagen de despliegue en diferentes entornos sin modificar el bytecode. Spring Boot resuelve las propiedades en un orden de precedencia bien definido donde las fuentes externas anulan las internas.

```yaml
# application.yml - configuracion base
spring:
  application:
    name: mi-aplicacion
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  threads:
    virtual:
      enabled: true

server:
  port: 8080
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
```

```yaml
# application-produccion.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

  cache:
    type: redis
    redis:
      host: ${REDIS_HOST}
      port: 6379

logging:
  level:
    root: WARN
    com.ejemplo: INFO
```

## Perfiles por entornos

Los perfiles permiten activar configuraciones especificas segun el entorno de ejecucion. Cada archivo `application-{perfil}.yml` contiene las propiedades que difieren de la configuracion base para ese entorno.

```yaml
# application-local.yml - Desarrollo local
spring:
  datasource:
    url: jdbc:h2:mem:localdb
    driver-class-name: org.h2.Driver
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop
  h2:
    console:
      enabled: true
      path: /h2-console

logging:
  level:
    com.ejemplo: DEBUG
    org.springframework.web: DEBUG

# application-testing.yml - Entorno de pruebas
spring:
  datasource:
    url: jdbc:postgresql://test-db:5432/testdb
  jpa:
    hibernate:
      ddl-auto: validate

# application-produccion.yml - Produccion
spring:
  jpa:
    show-sql: false
  threads:
    virtual:
      enabled: true

logging:
  level:
    root: WARN
```

```bash
# Activar perfil via variable de entorno
export SPRING_PROFILES_ACTIVE=produccion
java -jar mi-aplicacion.jar

# O via argumento de linea de comandos
java -jar mi-aplicacion.jar --spring.profiles.active=produccion
```

## @ConfigurationProperties

`@ConfigurationProperties` vincula grupos de propiedades del entorno a clases Java tipadas, proporcionando validacion, autocompletado en IDEs y una estructura clara de la configuracion.

```java
@ConfigurationProperties(prefix = "app.pagination")
@Validated
public class PaginationProperties {

    @NotNull
    @Min(1)
    @Max(100)
    private Integer defaultSize = 20;

    @NotNull
    @Min(1)
    @Max(500)
    private Integer maxSize = 100;

    private boolean allowCustomSize = true;

    private Map<String, Integer> defaultsByEntity = new HashMap<>();

    // Getters y setters
}
```

```java
@Configuration
@EnableConfigurationProperties(PaginationProperties.class)
public class PaginationConfig {

    @Bean
    public PaginationInterceptor paginationInterceptor(
            PaginationProperties properties) {
        return new PaginationInterceptor(properties);
    }
}
```

## Evitar valores magicos

Los valores magicos (cadenas de texto, numeros o constantes hardcodeadas) dificultan el mantenimiento y la comprension del codigo. Todos los valores que pueden cambiar entre entornos o que tienen significado de negocio deben externalizarse.

```java
// MAL: valores magicos en el codigo
public class PedidoService {
    public List<Pedido> recientes() {
        return repository.findByFechaAfter(
            LocalDateTime.now().minusDays(30)); // 30 es magico
    }
}

// BIEN: configuracion externalizada
@Service
public class PedidoService {

    private final PedidoProperties properties;

    public PedidoService(PedidoProperties properties) {
        this.properties = properties;
    }

    public List<Pedido> recientes() {
        return repository.findByFechaAfter(
            LocalDateTime.now().minus(properties.getDiasRecientes()));
    }
}
```

```java
@ConfigurationProperties(prefix = "app.pedidos")
public class PedidoProperties {
    private int diasRecientes = 30;
    private int maxItems = 50;
    private BigDecimal montoMaximo = new BigDecimal("10000.00");
    // Getters y setters
}
```

La gestion centralizada de configuracion evita la duplicacion de valores, facilita los cambios de comportamiento sin recompilacion y permite a los equipos operativos ajustar parametros sin intervencion del equipo de desarrollo. Cada valor magico en el codigo es una oportunidad para un error de configuracion que solo se descubrira en produccion.

