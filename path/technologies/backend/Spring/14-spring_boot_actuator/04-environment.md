# Environment

El endpoint `/actuator/env` expone el entorno completo de la aplicacion, incluyendo todas las fuentes de propiedades, variables de entorno del sistema, propiedades del sistema Java y todas las propiedades de Spring resueltas. Es una herramienta esencial para diagnosticar problemas de configuracion en entornos de produccion donde las propiedades pueden provenir de multiples fuentes y no es inmediato determinar cual esta tomando efecto.

## Property sources

Spring Boot agrega las fuentes de propiedades en un orden de prioridad bien definido. El endpoint `/env` lista todas estas fuentes y el valor final de cada propiedad, permitiendo rastrear desde donde proviene cada una.

```bash
curl http://localhost:8080/actuator/env | jq '.propertySources[] | .name'
```

```json
{
  "propertySources": [
    {"name": "systemProperties"},
    {"name": "systemEnvironment"},
    {"name": "applicationConfig: [classpath:/application.yml]"},
    {"name": "applicationConfig: [classpath:/application-produccion.yml]"},
    {"name": "applicationConfig: [classpath:/application-local.yml]"}
  ]
}
```

Las fuentes de propiedades se resuelven en el siguiente orden de prioridad (de mayor a menor):

1. Argumentos de linea de comandos (`--propiedad=valor`)
2. Variables de sistema Java (`-Dpropiedad=valor`)
3. Variables de entorno del SO (`PROPIEDAD=valor`)
4. Archivos `application-{profile}.yml` del perfil activo
5. Archivo `application.yml` principal
6. Propiedades por defecto de Spring Boot

```yaml
management:
  endpoint:
    env:
      keys-to-sanitize: password,secret,token,credential
  endpoints:
    web:
      exposure:
        include: env
```

> El endpoint `/env` expone informacion sensible del sistema (credenciales, tokens, claves de API). En produccion, se recomienda restringir su acceso con autenticacion y autorizacion, o sanitizar las claves sensibles usando `keys-to-sanitize`.

## Sanitizacion de propiedades

Para proteger informacion sensible, Spring Boot sanitiza automaticamente propiedades cuyos nombres contienen palabras clave como `password`, `secret`, `key` o `token`. Los valores sanitizados se reemplazan por `***` en la respuesta.

```yaml
management:
  endpoint:
    env:
      keys-to-sanitize: password,secret,token,credential,apikey
      # Expresiones regex tambien son validas
      # keys-to-sanitize: ".*password.*,.*secret.*"
```

Las expresiones regex permiten un control granular sobre que propiedades se sanitizan. En un entorno de desarrollo, la sanitizacion puede desactivarse para facilitar la depuracion.

## ConfiguracionProperties binding

`@ConfigurationProperties` es el mecanismo preferido para vincular propiedades del entorno a clases Java tipadas. A diferencia de `@Value`, que inyecta propiedades individuales, `@ConfigurationProperties` permite agrupar propiedades relacionadas en un objeto de configuracion con validacion y anidamiento.

```java
@ConfigurationProperties(prefix = "app.datasource")
public class DatasourceProperties {

    private String url;
    private String username;
    private String password;
    private int maximumPoolSize = 10;
    private int minimumIdle = 5;
    private Duration connectionTimeout = Duration.ofSeconds(30);
    private Duration idleTimeout = Duration.ofMinutes(10);
    private Duration maxLifetime = Duration.ofMinutes(30);

    // Getters y setters
}
```

```yaml
app:
  datasource:
    url: jdbc:postgresql://localhost:5432/mibd
    username: admin
    password: ${DB_PASSWORD:secreto}
    maximum-pool-size: 20
    minimum-idle: 5
    connection-timeout: 10s
    idle-timeout: 30m
    max-lifetime: 60m
```

La ventaja de `@ConfigurationProperties` sobre `@Value` es que soporta anidamiento, colecciones, validacion con Bean Validation y renombramiento automatico de propiedades (kebab-case en YAML a camelCase en Java).

```java
@Configuration
@EnableConfigurationProperties(DatasourceProperties.class)
public class DataSourceConfig {

    @Bean
    public DataSource dataSource(DatasourceProperties props) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(props.getUrl());
        config.setUsername(props.getUsername());
        config.setPassword(props.getPassword());
        config.setMaximumPoolSize(props.getMaximumPoolSize());
        config.setMinimumIdle(props.getMinimumIdle());
        return new HikariDataSource(config);
    }
}
```

El binding con `@ConfigurationProperties` valida que las propiedades requeridas esten presentes y que los tipos sean correctos. Si una propiedad tiene un tipo incompatible (ej: un `Duration` con valor `"abc"`), Spring lanza un `BindException` durante el arranque, detectando problemas de configuracion antes de que la aplicacion empiece a procesar trafico.

