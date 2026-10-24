# Configuration

La configuracion en Spring determina como el contenedor IoC descubre, crea y ensambla los beans. Spring ofrece multiples mecanismos de configuracion que van desde la deteccion automatica de componentes hasta la definicion explicita de beans via codigo, pasando por la carga de propiedades externas. La eleccion del mecanismo de configuracion impacta directamente en la mantenibilidad, testabilidad y flexibilidad de la aplicacion.

En la practica moderna con Spring Boot, la mayoria de las aplicaciones utilizan una combinacion de deteccion automatica de componentes y configuracion basada en anotaciones. Sin embargo, comprender todos los mecanismos disponibles es esencial para resolver situaciones donde la configuracion automatica no es suficiente.

## @Component y deteccion automatica

La anotacion `@Component` marca una clase como candidata a ser gestionada por el contenedor IoC. Cuando el contenedor escanea el classpath, detecta todas las clases anotadas con `@Component` (o sus estereotipos) y las registra como beans. El nombre del bean se deriva del nombre de la clase (primera letra en minuscula).

```java
@Component
public class CalculadorImpuestos {

    public double calcular(double base) {
        return base * 0.21;
    }
}
```

Spring proporciona estereotipos especiales de `@Component` para indicar la capa a la que pertenece el bean. Estos estereotipos no tienen comportamiento diferencial respecto a `@Component` en terminos de deteccion, pero comunican intencion y permiten filtros de escaneo especificos:

- `@Service`: capa de negocio. Indica que la clase contiene logica de negocio y orquesta las llamadas a repositorios y otros servicios.
- `@Repository`: capa de acceso a datos. Ademas de marcar la clase como componente, Spring aplica un proxy que traduce las checked exceptions de persistencia (como `PersistenceException`) a unchecked exceptions del hierarchy `DataAccessException`.
- `@Controller`: capa web (Spring MVC). Marca la clase como controller que maneja peticiones HTTP. Works con `@RequestMapping` y sus variantes.
- `@RestController`: combinacion de `@Controller` y `@ResponseBody`. Todos los metodos devuelven directamente el objeto serializado (tipicamente JSON) sin pasar por un ViewResolver.

```java
@Service
public class ServicioCliente {

    private final RepositorioCliente repositorio;

    public ServicioCliente(RepositorioCliente repositorio) {
        this.repositorio = repositorio;
    }

    public Cliente obtenerPorId(Long id) {
        return repositorio.findById(id)
            .orElseThrow(() -> new ClienteNoEncontradoException(id));
    }
}

@Repository
public class RepositorioCliente {

    private final JdbcTemplate jdbcTemplate;

    public RepositorioCliente(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Optional<Cliente> findById(Long id) {
        String sql = "SELECT id, nombre, email FROM clientes WHERE id = ?";
        List<Cliente> resultados = jdbcTemplate.query(sql, new ClienteRowMapper(), id);
        return resultados.stream().findFirst();
    }
}
```

> En Spring 5.3 y posteriores, `@Repository`, `@Service` y `@Controller` son meta-anotaciones de `@Component`. Esto significa que cualquier anotacion personalizada que sea meta-anotada con `@Component` sera detectada por el escaneo de componentes.

## @ComponentScan y escaneo de componentes

`@ComponentScan` configura el escaneo automatico de componentes en el classpath. El contenedor busca clases anotadas con `@Component` (y sus estereotipos) en los paquetes especificados y las registra como beans.

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = CustomAnnotation.class
    ),
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ASSIGNABLE_TYPE,
        classes = ServicioLegacy.class
    )
)
public class AppConfig {
    // El escaneo busca en "com.example" y todos sus subpaquetes
}
```

`@SpringBootApplication` incluye `@ComponentScan` implicitamente con el paquete base de la clase principal. Si la clase principal esta en `com.example.app`, el escaneo buscara en `com.example.app` y todos sus subpaquetes.

```java
@SpringBootApplication
// Equivalente a:
// @Configuration
// @EnableAutoConfiguration
// @ComponentScan(basePackages = "com.example.app")
public class MiAplicacion {
    public static void main(String[] args) {
        SpringApplication.run(MiAplicacion.class, args);
    }
}
```

El escaneo de componentes es selectivo por defecto: solo detecta clases con anotaciones de tipo `@Component`. No detecta clases que implementen interfaces o extiendan clases sin la anotacion correspondiente. Este comportamiento puede modificarse via `includeFilters` y `excludeFilters`.

## @Configuration y @Bean

`@Configuration` indica que una clase es una fuente de definiciones de beans. A diferencia de la deteccion automatica, donde el contenedor detecta y crea beans implicitamente, `@Configuration` permite definir beans de forma explicita con control total sobre la creacion y configuracion.

```java
@Configuration
public class ConfiguracionCache {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .build());
        return manager;
    }

    @Bean
    public ServicioCache servicioCache(CacheManager cacheManager) {
        return new ServicioCache(cacheManager);
    }
}
```

Las clases anotadas con `@Configuration` son proxyedas por CGLIB. Esto garantiza que las llamadas a metodos `@Bean` dentro de la misma clase retornen la misma instancia (singleton), no creen una nueva. Este comportamiento se conoce como "full mode" de `@Configuration`.

```java
@Configuration
public class ConfiguracionEjemplo {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // CGLIB proxy garantiza singleton
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }

    // Sin el proxy CGLIB, serviceA() crearia una nueva instancia
    // de ServiceB cada vez que se invocara internamente
}
```

Si se necesita desactivar el proxy CGLIB (por ejemplo, para clases que no necesitan inter-bean references), se usa `proxyBeanMethods = false`:

```java
@Configuration(proxyBeanMethods = false)
public class ConfiguracionLite {
    // Sin proxy CGLIB, mas rapido de iniciar
    // pero las llamadas a metodos @Bean crean nuevas instancias
}
```

> `proxyBeanMethods = false` (configuracion "lite") es la recomendacion de Spring Boot 3.x para clases de configuracion donde no se necesita referenciar otros beans de la misma clase. Reduce el consumo de memoria y el tiempo de inicio.

## @PropertySource y @Value

`@PropertySource` carga propiedades desde un archivo externo (tipicamente `.properties` o `.yaml`) y las hace disponibles en el `Environment` del contenedor. `@Value` inyecta el valor de una propiedad especifica en un campo, parametro o metodo.

```java
@Configuration
@PropertySource("classpath:database.properties")
public class ConfiguracionBaseDatos {

    @Value("${db.url}")
    private String url;

    @Value("${db.username}")
    private String username;

    @Value("${db.password}")
    private String password;

    @Value("${db.pool.max-size:10}")
    private int poolMaxSize; // Valor por defecto si no existe la propiedad

    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url(url)
            .username(username)
            .password(password)
            .build();
    }
}
```

En Spring Boot, `@PropertySource` es innecesario para el archivo `application.properties` o `application.yml`, ya que se carga automaticamente. Solo se necesita `@PropertySource` para archivos de propiciones adicionales.

```yaml
# application.yml
app:
  servicio:
    nombre: "Mi Aplicacion"
    timeout: 30
    retry:
      max-attempts: 3
```

```java
@Service
public class ServicioExterno {

    @Value("${app.servicio.nombre}")
    private String nombre;

    @Value("${app.servicio.timeout}")
    private int timeout;

    @Value("${app.servicio.retry.max-attempts}")
    private int maxAttempts;
}
```

## @Import y @ImportResource

`@Import` permite importar definiciones de beans desde otra clase `@Configuration` o desde una clase que implemente `ImportSelector` o `ImportBeanDefinitionRegistrar`. Es la forma de modularizar la configuracion en multiples archivos sin necesidad de `@ComponentScan`.

```java
@Configuration
@Import({ConfiguracionSeguridad.class, ConfiguracionCache.class})
public class AppConfig {
    // Importa las definiciones de beans de las clases indicadas
}

@Configuration
public class ConfiguracionSeguridad {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

`@ImportResource` carga configuracion XML legacy dentro de una aplicacion basada en anotaciones. Es util para migraciones incrementales donde parte de la configuracion aun esta en XML.

```java
@Configuration
@ImportResource("classpath:legacy-beans.xml")
public class AppConfig {
    // Los beans definidos en legacy-beans.xml
    // se cargan junto con los de esta clase
}
```

## Relacion entre anotaciones de configuracion

Las anotaciones de configuracion de Spring forman un ecosistema coherente donde cada una tiene un proposito especifico. `@Configuration` y `@Bean` definen beans de forma explicita. `@Component` (y sus estereotipos) permiten la deteccion automatica. `@ComponentScan` activa el escaneo. `@PropertySource` y `@Value` gestionan propiedades externas. `@Import` y `@ImportResource` modularizan la configuracion.

En la practica, una aplicacion Spring Boot moderna utiliza principalmente `@SpringBootApplication` (que incluye `@ComponentScan` y `@EnableAutoConfiguration`), `@Configuration` con `@Bean` para definiciones explicitas, y `@Value` o `@ConfigurationProperties` para propiedades externas. Las demas anotaciones se usan en escenarios especificos o durante migraciones de configuraciones legacy.

> La tendencia en Spring 6 y Spring Boot 3.x es reducir la cantidad de configuracion manual. La auto-configuracion, los properties binded a clases tipadas via `@ConfigurationProperties` y el uso de GraalVM native images estan desplazando progresivamente la necesidad de `@Configuration` explicita en muchos casos. Sin embargo, comprender estos mecanismos sigue siendo fundamental para resolver problemas y personalizar el comportamiento cuando las convenciones no son suficientes.

