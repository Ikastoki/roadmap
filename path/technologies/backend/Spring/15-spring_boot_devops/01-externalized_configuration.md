# Externalized Configuration

La configuracion externa es el mecanismo que permite separar los valores de configuracion del codigo fuente, de modo que la misma aplicacion pueda ejecutarse en diferentes entornos (desarrollo, staging, produccion) sin modificar una sola linea de codigo. Spring Boot implementa un sistema de jerarquia de fuentes de configuracion que determina el orden de precedencia cuando una propiedad esta definida en multiples lugares.

## Prioridades de configuracion

Spring Boot resuelve las propiedades siguiendo un orden estricto de precedencia, donde las fuentes con mayor prioridad anulan las de menor prioridad. El orden completo de mayor a menor prioridad es:

1. Argumentos de linea de comandos (`--server.port=9090`)
2. Propiedades de sistema Java (`-Dserver.port=9090`)
3. Variables de entorno del SO (`SERVER_PORT=9090`)
4. Archivo `application-{profile}.properties` o `.yml` del perfil activo
5. Archivo `application.properties` o `.yml` principal
6. `@PropertySource` en el codigo de la aplicacion
7. Propiedades por defecto de Spring Boot (`SpringApplication.setDefaultProperties`)

```yaml
# application.yml (perfil por defecto)
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mibd
    username: admin
    password: dev123

---
# application-produccion.yml (sobreescribe las anteriores)
spring:
  config:
    activate:
      on-profile: produccion
  datasource:
    url: jdbc:postgresql://db-produccion:5432/mibd
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
server:
  port: 443
```

## Perfiles especificos

Los perfiles permiten definir configuraciones alternativas que se activan segun el entorno de ejecucion. Un archivo `application-{perfil}.yml` se carga cuando ese perfil esta activo, sobreescribiendo las propiedades definidas en `application.yml`.

```java
@SpringBootApplication
public class MiAplicacion {

    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MiAplicacion.class);
        app.setAdditionalProfiles("produccion");
        app.run(args);
    }
}
```

```yaml
# application-desarrollo.yml
spring:
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop
logging:
  level:
    com.ejemplo: DEBUG

# application-testing.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop

# application-produccion.yml
spring:
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate
logging:
  level:
    com.ejemplo: INFO
    root: WARN
```

## Inyeccion con @Value

`@Value` inyecta una propiedad individual directamente en un campo o parametro de metodo. Soporta valores por defecto con la sintaxis `${propiedad:valor_defecto}` y expresiones SpEL (Spring Expression Language).

```java
@Component
public class ConfiguracionApp {

    @Value("${app.nombre:Mi Aplicacion}")
    private String nombre;

    @Value("${app.tamano-pagina:20}")
    private int tamanoPagina;

    @Value("${app.features.beta:false}")
    private boolean betaHabilitado;

    @Value("#{${app.colores} ?: {'#FF0000','#00FF00'}}")
    private List<String> colores;

    @Value("${app.timeout:30s}")
    private Duration timeout;
}
```

> `@Value` no soporta validacion con Bean Validation y no genera metadatos de autocompletado para IDEs. Para configuraciones complejas con anidamiento y validacion, `@ConfigurationProperties` es preferible.

## Binding con @ConfigurationProperties

`@ConfigurationProperties` vincula un grupo de propiedades a un objeto POJO con soporte completo para anidamiento, coleccioness, tipos complejos y validacion. Es el mecanismo recomendado para configuraciones que involucran multiples propiedades relacionadas.

```java
@ConfigurationProperties(prefix = "app.pagination")
public class PaginationProperties {

    @NotNull
    private Integer defaultSize;

    @Max(100)
    private Integer maxSize;

    private boolean allowCustomSize = true;

    private Map<String, Integer> defaultsByEntity = new HashMap<>();

    // Getters y setters
}
```

```yaml
app:
  pagination:
    default-size: 20
    max-size: 100
    allow-custom-size: true
    defaults-by-entity:
      usuario: 15
      producto: 25
      pedido: 10
```

El binding con `@ConfigurationProperties` tiene ventajas significativas sobre `@Value`: soporta validacion automatica con `@Validated`, genera metadatos de autocompletado para IDEs con `spring-boot-configuration-processor`, y permite renombramiento automatico de propiedades (kebab-case en YAML a camelCase en Java). Ademas, los cambios en las propiedades se propagan inmediatamente cuando se usa `@ConfigurationProperties` con `@RefreshScope`.

