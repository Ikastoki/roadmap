# Config Server

Spring Cloud Config Server centraliza la gestion de configuracion de multiples microservicios en un unico repositorio accesible a traves de una API REST. En arquitecturas de microservicios, donde decenas de servicios necesitan configuraciones consistentes y versionadas, mantener archivos de configuracion locales en cada repositorio genera una friccion operativa significativa: cambios en propiedades compartidas requieren modificaciones en multiples repositorios y despliegues simultaneos.

El Config Server resuelve este problema exponiendo la configuracion de todos los servicios a traves de un endpoint HTTP. Los clientes obtienen su configuracion al iniciar la aplicacion, identificandose por el nombre de la aplicacion y el perfil activo. Esto permite mantener una fuente unica de verdad para toda la configuracion del ecosistema de microservicios.

## Arquitectura del sistema

```
Git Repository (configuraciones)
        |
        v
  Config Server (API REST)
        |
   +----+----+----+
   |    |    |    |
   v    v    v    v
 svc1 svc2 svc3 svc4  (clientes que obtienen su configuracion)
```

## Spring Cloud Config Server

El Config Server se despliega como una aplicacion Spring Boot independiente que se conecta a un backend de almacenamiento y expone la configuracion a traves de endpoints predefinidos.

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/company/config-repo
          default-label: main
          search-paths:
            - '{application}'
            - 'shared'
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          clone-on-start: true
          refresh-rate: 300
```

## Git Backend

Git es el backend recomendado para el Config Server. Cada microservicio tiene su propia carpeta en el repositorio de configuracion, y los cambios se versionan automaticamente con el historial de commits de Git. Esto proporciona auditabilidad completa de los cambios de configuracion, la capacidad de reverts instantaneos y branching para probar configuraciones antes de promocionarlas a produccion.

La estructura tipica del repositorio de configuracion sigue el patron `{application}/{profile}/{application}-{profile}.properties` o su equivalente YAML.

```
config-repo/
  order-service/
    application.yml
    application-dev.yml
    application-prod.yml
  inventory-service/
    application.yml
    application-dev.yml
  shared/
    common.yml
```

```yaml
# config-repo/order-service/application.yml
spring:
  datasource:
    url: jdbc:postgresql://db-server:5432/orders
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: validate

app:
  order:
    timeout: 30
    retry-attempts: 3
```

El Config Server compone la configuracion consultando multiples fuentes: primero la configuracion especifica del servicio, luego los perfiles activos, y finalmente los archivos compartidos. El orden de precedencia garantiza que la configuracion mas especifica sobreescribe la generica.

## Native Backend

El backend nativo utiliza el sistema de archivos local en lugar de Git como fuente de configuracion. Es util para entornos de desarrollo o despliegues donde el acceso a un repositorio Git no es viable.

```yaml
spring:
  cloud:
    config:
      server:
        native:
          search-locations:
            - file:///opt/config/{application}
            - classpath:/config/{application}
          fail-on-missing-location: false
```

> El backend nativo no proporciona versionado ni auditabilidad de cambios. Utilizarlo exclusivamente en entornos de desarrollo o como alternativa temporal cuando el repositorio Git no esta disponible.

## Spring Cloud Config Client

Los microservicios que consumen la configuracion del Config Server incorporan el cliente de Spring Cloud Config. Al iniciar, el cliente contacta al Config Server, obtiene su configuracion y la integra en el Environment de la aplicacion como si fuera un archivo de configuracion local.

```yaml
spring:
  application:
    name: order-service
  profiles:
    active: production
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        max-interval: 16000
        multiplier: 1.5
      label: main
```

La propiedad `fail-fast: true` obliga a la aplicacion a fallar si no puede conectar con el Config Server al iniciar. Sin esta propiedad, la aplicacion arranca con la configuracion local, lo que puede generar comportamientos inesperados en produccion.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

## @RefreshScope

La anotacion `@RefreshScope` habilita la actualizacion dinamica de beans sin reiniciar la aplicacion. Cuando el Config Server recibe un evento de actualizacion, los beans anotados con `@RefreshScope` se destruyen y recrean con los nuevos valores de configuracion. Esto permite cambiar propiedades como timeouts, feature flags o parametros de negocio en tiempo real.

```java
@Service
@RefreshScope
public class OrderService {

    @Value("${app.order.timeout:30}")
    private int orderTimeout;

    @Value("${app.order.max-retry:3}")
    private int maxRetries;

    public int getOrderTimeout() {
        return orderTimeout;
    }
}
```

```bash
# Forzar refresh de la configuracion en el cliente
curl -X POST http://order-service:8080/actuator/refresh
```

> `@RefreshScope` solo actualiza beans que dependen directamente de `@Value` o `@ConfigurationProperties`. Los valores que se leen una vez en el constructor o en un `@PostConstruct` no se actualizan hasta que el bean se recrea.

## Spring Cloud Bus for Refresh

Spring Cloud Bus extiende el patron de refresh utilizando un sistema de mensajeria (RabbitMQ o Kafka) para propagar eventos de actualizacion de configuracion a todos los servicios simultaneamente. Sin Cloud Bus, cada servicio debe recibir un evento de refresh individual. Con Cloud Bus, un unico evento en el Config Server notifica a todos los clientes conectados al bus.

```yaml
# Config Server
spring:
  rabbitmq:
    host: rabbitmq-server
    port: 5672
    username: guest
    password: guest
  cloud:
    bus:
      enabled: true
      refresh:
        enabled: true
      trace:
        enabled: true
```

```bash
# Notificar actualizacion a todos los servicios a traves del bus
curl -X POST http://config-server:8888/actuator/busrefresh
```

```
Config Server ---[abbitMQ/Kafka]---> Service A (refresh)
                              |----> Service B (refresh)
                              |----> Service C (refresh)
```

La combinacion de Config Server con Git, RefreshScope y Cloud Bus crea un sistema de gestion de configuracion completo que permite actualizaciones centralizadas, versionadas y propagadas de forma automatica a todos los microservicios del ecosistema.

