# Profiles

Los profiles en Spring proporcionan un mecanismo para segmentar la configuracion de la aplicacion segun el entorno de ejecucion. En lugar de mantener archivos de configuracion separados para desarrollo, testing y produccion, o de usar variables de entorno con condicionales, los profiles permiten definir beans, propiedades y configuraciones que solo se activan cuando un profile especifico esta activo. Esto garantiza que cada entorno reciba exactamente la configuracion que necesita sin duplicacion ni condicionales en el codigo.

Un profile es simplemente un nombre logico (tipicamente una cadena como "dev", "test", "prod" o "staging") que se asocia a configuraciones, beans y propiedades. Cuando la aplicacion arranca, se especifican los profiles activos, y el contenedor solo carga las configuraciones y beans correspondientes. Las configuraciones que no pertenecen a un profile activo se ignoran completamente.

## @Profile en beans y configuraciones

La anotacion `@Profile` se aplica a clases `@Configuration` o a metodos `@Bean` para indicar que solo deben cargarse cuando un profile especifico (o alguno de ellos) este activo.

```java
@Configuration
@Profile("dev")
public class ConfiguracionDesarrollo {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("schema.sql")
            .addScript("data-dev.sql")
            .build();
    }

    @Bean
    public MailSender mailSender() {
        return new DummyMailSender(); // Sin envio real en desarrollo
    }
}

@Configuration
@Profile("prod")
public class ConfiguracionProduccion {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(env.getProperty("db.url"));
        config.setUsername(env.getProperty("db.username"));
        config.setPassword(env.getProperty("db.password"));
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }

    @Bean
    public MailSender mailSender() {
        return new JavaMailSenderImpl(); // Envio real en produccion
    }
}
```

```java
@Configuration
public class ConfiguracionComun {

    @Bean
    @Profile("dev")
    public CommandLineRunner datosPrueba(RepositorioPedido repositorio) {
        return args -> {
            repositorio.save(new Pedido("Pedido de prueba 1"));
            repositorio.save(new Pedido("Pedido de prueba 2"));
        };
    }

    @Bean
    @Profile("!prod")
    public ServicioDepuracion servicioDepuracion() {
        return new ServicioDepuracionDetallada();
    }
}
```

> Los profiles soportan expresiones logicas. `@Profile("dev")` se activa solo con el profile "dev". `@Profile("!prod")` se activa cuando "prod" NO esta activo. `@Profile({"dev", "test"})` se activa cuando cualquiera de los dos esta activo. La expresion mas compleja se resuelve con `@Profile("dev & !staging")`.

## Configuracion de profiles activos

Los profiles se activan de multiples formas, ordenadas por prioridad:

**Variables de entorno**: la forma mas comun en contenedores Docker y Kubernetes.

```bash
export SPRING_PROFILES_ACTIVE=prod,docker
java -jar mi-aplicacion.jar
```

**Propiedades del sistema**: util para desarrollo local.

```bash
java -jar mi-aplicacion.jar -Dspring.profiles.active=dev
```

**Archivo application.yml**: definicion por defecto.

```yaml
spring:
  profiles:
    active: dev  # Profile por defecto, sobreescrito por entorno
```

**Configuracion programatica**: para casos donde el profile depende de logica de negocio.

```java
@SpringBootApplication
public class MiAplicacion {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MiAplicacion.class);
        app.setAdditionalProfiles("custom");
        app.run(args);
    }
}
```

## Archivos de configuracion especificos por profile

Spring Boot soporta archivos de configuracion especificos por profile con un mecanismo de nombres predecible. El archivo `application-{profile}.yml` (o `.properties`) se carga automaticamente cuando el profile correspondiente esta activo, y sus propiedades sobreescriben las del archivo principal.

```
src/main/resources/
  application.yml          # Configuracion base
  application-dev.yml      # Configuracion de desarrollo
  application-prod.yml     # Configuracion de produccion
  application-test.yml     # Configuracion de testing
```

```yaml
# application.yml
spring:
  datasource:
    driver-class-name: org.postgresql.Driver

app:
  log-level: INFO
  mail:
    host: localhost

---
# Configuracion de desarrollo
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver

app:
  log-level: DEBUG
  mail:
    host: localhost
    port: 1025  # MailHog / MailCatcher

---
# Configuracion de produccion
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://db-cluster:5432/app_prod
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

app:
  log-level: WARN
  mail:
    host: smtp.gmail.com
    port: 587
```

> En Spring Boot 3.x, el formato `spring.profiles.active` en YAML ha sido reemplazado por `spring.config.activate.on-profile`. El formato anterior sigue funcionando por compatibilidad, pero el nuevo es el recomendado.

## Activacion programatica

En algunos escenarios, el profile activo depende de condiciones que solo se conocen en tiempo de ejecucion, como la presencia de un archivo de configuracion, una variable de entorno especifica o una condicion de red.

```java
@Component
public class ActivadorProfiles implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(
            ConfigurableEnvironment environment,
            SpringApplication application) {

        String host = System.getenv().getOrDefault("HOSTNAME", "");

        if (host.startsWith("prod-")) {
            environment.addActiveProfile("prod");
        } else if (host.startsWith("staging-")) {
            environment.addActiveProfile("staging");
        } else {
            environment.addActiveProfile("dev");
        }

        if (Files.exists(Path.of("/etc/app/custom.properties"))) {
            environment.addActiveProfile("custom");
        }
    }
}
```

```properties
# META-INF/spring.factories (Spring Boot 2.x)
org.springframework.boot.env.EnvironmentPostProcessor=com.example.ActivadorProfiles

# META-INF/spring/org.springframework.boot.env.EnvironmentPostProcessor.imports (Spring Boot 3.x)
com.example.ActivadorProfiles
```

## Profile groups

Los profile groups permiten agrupar multiples profiles bajo un nombre unico, simplificando la gestion de configuraciones complejas donde un entorno requiere activar varios profiles simultaneamente.

```yaml
spring:
  profiles:
    group:
      production:
        - prod
        - metrics
        - security-hardened
        - external-services
      development:
        - dev
        - debug
        - local-database
```

```bash
# Activar el grupo "production" activa: prod, metrics, security-hardened, external-services
export SPRING_PROFILES_ACTIVE=production
```

Los profile groups son especialmente utiles en arquitecturas donde un entorno de produccion requiere configuracion de metricas (Micrometer/Prometheus), seguridad avanzada (Spring Security), servicios externos (Redis, Kafka) y logging estructurado (ELK stack), todo ello gestionado como un grupo coherente.

> Los profiles de Spring son un mecanismo de configuracion, no de aislamiento. Un bean inyectado desde un profile no activo simplemente no se crea, pero si otro bean depende de el y el perfil no esta activo, la aplicacion fallara al iniciar. Es importante que las dependencias entre profiles sean consistentes y que los beans de un perfil no dependan implicitamente de beans de otro perfil.

