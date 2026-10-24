# Logging de SQL

Depurar problemas de persistencia sin ver las sentencias SQL que genera el ORM es como trabajar a ciegas. El logging de SQL permite inspeccionar que consultas se ejecutan, con que parametros, cuanto tardan y si el numero de consultas es el esperado. Herramientas como P6Spy y DataSource-proxy llevan esta visibilidad un paso mas alla al interceptar la capa JDBC, mostrando incluso consultas que Hibernate no expone directamente.

## show-sql y format_sql

La configuracion mas basica en Spring Boot activa la impresion de las sentencias SQL generadas por Hibernate en la consola:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
```

- `show-sql: true` habilita la salida de cada sentencia SQL por `System.out`.
- `format_sql: true` aplica un formateo legible con saltos de linea y sangria.
- `use_sql_comments: true` anade comentarios indicando que operacion origino la consulta (ej. `/* insert Usuario */ insert into usuario ...`).

> `show-sql` usa `System.out` directamente, lo que dificulta su captura en logs estructurados y afecta al rendimiento en produccion. En entornos reales es preferible usar el logger de Hibernate.

## Hibernate logging

En lugar de `show-sql`, la configuracion recomendada para entornos de desarrollo y preproduccion utiliza los loggers de Hibernate sobre SLF4J:

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

- `org.hibernate.SQL: DEBUG` muestra cada sentencia SQL con placeholders `?`.
- `org.hibernate.type.descriptor.sql.BasicBinder: TRACE` registra los valores reales vinculados a cada parametro.

Para ver ademas los tiempos de ejecucion y el numero de filas devueltas:

```yaml
logging.level.org.hibernate.stat: DEBUG
```

Y habilitando las estadisticas de Hibernate:

```yaml
spring.jpa.properties.hibernate.generate_statistics: true
```

Esto produce una linea como la siguiente al finalizar cada sesion:

```
Session Metrics { 234234 nanoseconds spent preparing 1 JDBC statements; 
65434 nanoseconds spent executing 1 JDBC statements; 
0 nanoseconds spent executing 0 JDBC batches; 
0 nanoseconds spent performing 0 L2C puts; }
```

## P6Spy

P6Spy es una libreria que intercepta las llamadas JDBC y produce un log detallado con los valores reales de los parametros, tiempos de ejecucion y trazado de la pila de llamadas. Se integra configurando un datasource proxy.

Dependencia Maven:

```xml
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

Configuracion en `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:p6spy:postgresql://localhost:5432/mydb
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
```

Archivo `spy.properties` en `classpath`:

```properties
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
databaseDialectDateFormat=yyyy-MM-dd'T'HH:mm:ss
```

Con P6Spy cada consulta aparece con sus valores reales, facilitando la depuracion:

```
2026-07-28 10:32:15 | 3 ms | 1 | SELECT u.id, u.nombre FROM usuario u WHERE u.email = 'test@example.com'
```

> P6Spy introduce una sobrecarga en tiempo de ejecucion. Debe activarse solo en entornos de desarrollo o preproduccion, y jamas en produccion. Ademas es incompatible con ciertos pools de conexiones si no se configura correctamente el driver.

## DataSource-proxy

DataSource-proxy (de net.ttddyy) es una alternativa mas ligera que P6Spy. Permite envolver el DataSource real con un proxy que registra cada operacion JDBC mediante callbacks personalizados.

Dependencia:

```xml
<dependency>
    <groupId>net.ttddyy</groupId>
    <artifactId>datasource-proxy</artifactId>
    <version>1.10</version>
</dependency>
```

Configuracion programatica:

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        DataSource dataSource = DataSourceBuilder.create()
            .url("jdbc:postgresql://localhost:5432/mydb")
            .username("user")
            .password("pass")
            .build();

        return ProxyDataSourceBuilder
            .create(dataSource)
            .name("MyDS")
            .logQueryBySlf4j()
            .multiline()
            .build();
    }
}
```

La salida incluye el nombre del datasource, la consulta, los parametros y el tiempo de ejecucion:

```
Name:MyDS, Connection:5, Time:3, Success:True
Type:Prepared, Batch:False, QuerySize:1, BatchSize:0
Query:[SELECT u.id, u.nombre FROM usuario u WHERE u.email = ?]
Params:[(test@example.com)]
```

DataSource-proxy permite ademas anadir listeners personalizados para metricas o trazabilidad sin modificar el codigo del repositorio. Combinado con Spring Boot Actuator se puede exponer el tiempo medio de las consultas como metrica de observabilidad.
