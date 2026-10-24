# Apache DBCP2

Apache DBCP2 (Database Connection Pool 2) es una libreria de pool de conexiones JDBC mantenida por la Apache Software Foundation. Fue durante anos la opcion predeterminada en aplicaciones Java, hasta la llegada de HikariCP. Aunque su rendimiento es inferior al de HikariCP, sigue siendo una opcion viable en proyectos legacy o en entornos donde no se puede cambiar de dependencia.

## BasicDataSource

La clase central de DBCP2 es `BasicDataSource`, que implementa `javax.sql.DataSource` y proporciona toda la funcionalidad del pool.

```java
BasicDataSource ds = new BasicDataSource();
ds.setUrl("jdbc:postgresql://localhost:5432/mibd");
ds.setUsername("usuario");
ds.setPassword("contrasena");
ds.setDriverClassName("org.postgresql.Driver");

ds.setInitialSize(5);
ds.setMaxTotal(20);
ds.setMaxIdle(10);
ds.setMinIdle(2);
ds.setMaxWaitMillis(30000);
ds.setMinEvictableIdleTimeMillis(600000);
ds.setTimeBetweenEvictionRunsMillis(5000);
ds.setTestOnBorrow(true);
ds.setValidationQuery("SELECT 1");
```

## Configuracion de propiedades

DBCP2 se puede configurar mediante propiedades en Spring Boot, aunque desde Boot 2.x el pool predeterminado es HikariCP. Para usar DBCP2, es necesario excluir HikariCP e incluir DBCP2 explícitamente:

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-dbcp2</artifactId>
</dependency>
```

```properties
# Forzar el uso de DBCP2 en Spring Boot
spring.datasource.type=org.apache.commons.dbcp2.BasicDataSource
spring.datasource.dbcp2.initial-size=5
spring.datasource.dbcp2.max-total=20
spring.datasource.dbcp2.max-idle=10
spring.datasource.dbcp2.min-idle=2
spring.datasource.dbcp2.max-wait-millis=30000
spring.datasource.dbcp2.min-evictable-idle-time-millis=600000
spring.datasource.dbcp2.time-between-eviction-runs-millis=5000
spring.datasource.dbcp2.test-on-borrow=true
spring.datasource.dbcp2.validation-query=SELECT 1
```

### Parametros principales

| Propiedad | Descripcion | Valor recomendado |
|-----------|-------------|-------------------|
| `initialSize` | Conexiones creadas al iniciar el pool | 5 |
| `maxTotal` | Maximo de conexiones totales | 20 |
| `maxIdle` | Maximo de conexiones inactivas | 10 |
| `minIdle` | Minimo de conexiones inactivas | 2 |
| `maxWaitMillis` | Tiempo de espera maximo para obtener conexion | 30000 ms |
| `minEvictableIdleTimeMillis` | Tiempo de vida de conexiones inactivas | 600000 ms |
| `timeBetweenEvictionRunsMillis` | Intervalo entre ejecuciones del hilo de evacuacion | 5000 ms |
| `testOnBorrow` | Validar conexion al obtenerla del pool | `true` |
| `validationQuery` | Consulta de validacion | `SELECT 1` |

## Rendimiento

DBCP2 es significativamente mas lento que HikariCP debido a:
- Uso intensivo de sincronizacion con `synchronized` en metodos criticos.
- Validacion de conexiones en cada obtencion (`testOnBorrow=true`), que anade latencia.
- Algoritmo de prestamo basado en `ObjectPool` de Apache Pool2, que introduce overhead de abstraccion.

```text
Tiempo medio por operacion getConnection/close:

HikariCP  : 0.05 ms
DBCP2     : 0.35 ms
```

Para aplicaciones con baja concurrencia o donde la latencia no es critica, la diferencia es irrelevante. Para sistemas de alta concurrencia, la diferencia se acumula y HikariCP ofrece un rendimiento muy superior.

## Validacion de conexiones

A diferencia de HikariCP, DBCP2 valida las conexiones activamente. El parametro `testOnBorrow=true` ejecuta la `validationQuery` cada vez que se obtiene una conexion del pool.

```java
ds.setTestOnBorrow(true);
ds.setValidationQuery("SELECT 1");
```

Esto garantiza que la conexion devuelta es valida, pero anade una consulta SQL adicional por cada operacion. Alternativamente, se puede usar `testWhileIdle` para validar solo las conexiones inactivas periodicamente:

```java
ds.setTestWhileIdle(true);
ds.setTimeBetweenEvictionRunsMillis(5000);
ds.setValidationQuery("SELECT 1");
```

> `testWhileIdle` tiene menor impacto en el rendimiento que `testOnBorrow`, pero no garantiza que la conexion siga siendo valida en el momento de usarla. La eleccion depende del equilibrio entre seguridad y rendimiento que requiera la aplicacion.

## Ventajas de DBCP2

A pesar de su menor rendimiento, DBCP2 tiene ventajas en contextos especificos:
- API madura y estable con decadas de uso en produccion.
- Comportamiento predecible y bien documentado.
- Compatible con aplicaciones legacy que ya lo utilizan.
- Facil de configurar mediante propiedades estandar.

## Limitaciones

- Rendimiento inferior a HikariCP y Tomcat CP.
- Mayor consumo de memoria por conexion gestionada.
- Configuracion mas verbosa y con mas parametros.
- No tiene deteccion nativa de fugas de conexion.
- Las metricas no son tan completas como las de HikariCP.

Para proyectos nuevos, la recomendacion es usar HikariCP. DBCP2 solo tiene sentido en proyectos legacy donde el cambio de pool no este justificado o donde la dependencia este profundamente integrada en la configuracion.
