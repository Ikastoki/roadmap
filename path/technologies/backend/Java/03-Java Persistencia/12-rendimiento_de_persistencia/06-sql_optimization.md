# SQL optimization

Por muy eficiente que sea el mapeo ORM, toda operacion de persistencia termina traduciendose en sentencias SQL. Si esas sentencias no estan optimizadas, ningun nivel de abstraccion puede compensar el coste de una consulta ineficiente. La optimizacion SQL implica analizar el plan de ejecucion de las consultas, identificar cuellos de botella y garantizar que los indices adecuados estan disponibles.

## EXPLAIN ANALYZE

`EXPLAIN ANALYZE` es la herramienta fundamental para entender como ejecuta la base de datos una consulta. Muestra el plan de ejecucion, el coste estimado de cada operacion, el numero de filas procesadas y el tiempo real de ejecucion.

```sql
EXPLAIN ANALYZE
SELECT u.* FROM usuarios u
LEFT JOIN pedidos p ON p.usuario_id = u.id
WHERE u.activo = true
ORDER BY u.fecha_creacion DESC
LIMIT 20;
```

La salida tipica en PostgreSQL muestra:

```
Limit  (cost=1000.45..1200.30 rows=20 width=150) (actual time=0.45..0.50 rows=20 loops=1)
  ->  Sort  (cost=1000.45..1050.45 rows=20000 width=150) (actual time=0.45..0.48 rows=20 loops=1)
        Sort Key: u.fecha_creacion DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Hash Left Join  (cost=50.20..800.10 rows=20000 width=150) (actual time=0.20..0.35 rows=20000 loops=1)
              Hash Cond: (u.id = p.usuario_id)
              ->  Seq Scan on usuarios u  (cost=0.00..400.00 rows=20000 width=100) (actual time=0.01..0.10 rows=20000 loops=1)
                    Filter: activo = true
              ->  Hash  (cost=30.00..30.00 rows=1000 width=8) (actual time=0.15..0.15 rows=1000 loops=1)
                    ->  Seq Scan on pedidos p  (cost=0.00..30.00 rows=1000 width=8) (actual time=0.01..0.05 rows=1000 loops=1)
```

Los indicadores clave a observar son:

`Seq Scan` indica un escaneo secuencial de la tabla. Aparece cuando no hay un indice utilizable. En tablas grandes, un Seq Scan puede ser acceptable si la consulta devuelve un porcentaje alto de filas (mas del 10-20%). Para consultas selectivas, es una senal de alerta.

`actual time` muestra el tiempo real de ejecucion de cada nodo. El primer valor es el tiempo hasta la primera fila, el segundo es el tiempo hasta la ultima fila.

`rows` estimado vs `actual rows`: una discrepancia grande indica que el optimizador tiene estadisticas desactualizadas, lo que puede llevar a elegir un plan de ejecucion suboptimo.

```sql
-- Actualizar estadisticas
ANALYZE usuarios;
ANALYZE pedidos;
```

## Identificar queries lentas

En produccion, no es viable ejecutar `EXPLAIN ANALYZE` manualmente para cada consulta. La base de datos debe configurarse para registrar automaticamente las consultas lentas:

```properties
# PostgreSQL
log_min_duration_statement=1000  # ms

# MySQL
slow_query_log=1
long_query_time=1  # segundos

# Hibernate (para depuracion, no produccion)
spring.jpa.properties.hibernate.generate_statistics=true
```

Con `hibernate.generate_statistics=true`, Hibernate registra metricas detalladas de cada operacion: numero de consultas, tiempo de ejecucion, entidades cargadas en el cache y mas.

```yaml
logging:
  level:
    org.hibernate.stat: DEBUG
    org.hibernate.SQL: DEBUG
```

Los logs de Hibernate Statistics muestran informacion como:

```
Session Metrics {
    113485 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    2541710 nanoseconds spent preparing 25 JDBC statements;
    75831481 nanoseconds spent executing 25 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
}
```

## Analisis del plan de consultas JPA/Hibernate

Para analizar las consultas que genera Hibernate, se debe habilitar el log de SQL y capturar las sentencias:

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

Con `BasicBinder=TRACE`, los valores de los parametros bindeados tambien se muestran, permitiendo copiar la consulta completa con valores para ejecutar `EXPLAIN ANALYZE` manualmente.

Muchas herramientas de monitoreo (p6spy, datasource-proxy) interceptan las consultas JDBC y proporcionan informacion adicional como tiempo de ejecucion y stack trace del punto de llamada:

```xml
<dependency>
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>p6spy-spring-boot-starter</artifactId>
    <version>1.9.0</version>
</dependency>
```

```properties
decorator.datasource.p6spy.enable-logging=true
decorator.datasource.p6spy.log-format=executionTime:%(executionTime) | connection%(connectionId) | sql:%(sqlSingleLine)
```

## Indices

Los indices son el mecanismo principal para acelerar las consultas. Un indice bien disenado puede reducir el tiempo de una consulta de segundos a milisegundos. La regla fundamental es: crear indices para las columnas que aparecen en las clausulas `WHERE`, `JOIN` y `ORDER BY`.

```sql
-- Indice para busqueda por email
CREATE INDEX idx_usuarios_email ON usuarios (email);

-- Indice compuesto para filtros habituales
CREATE INDEX idx_usuarios_activo_fecha ON usuarios (activo, fecha_creacion DESC);

-- Indice para clave foranea (mejora JOINs)
CREATE INDEX idx_pedidos_usuario_id ON pedidos (usuario_id);
```

Los indices compuestos son especialmente potentes: un indice sobre `(activo, fecha_creacion DESC)` puede servir simultaneamente para filtrar por `activo`, ordenar por `fecha_creacion`, y si solo se necesitan esas columnas, incluso como covering index (indice de cubrimiento, que evita acceder a la tabla).

```sql
-- Indice de cubrimiento: contiene todas las columnas necesarias para la consulta
CREATE INDEX idx_usuarios_activo_fecha_covering ON usuarios (activo, fecha_creacion DESC) INCLUDE (nombre, email);
```

> Los indices no son gratuitos. Cada indice ralentiza las operaciones de escritura (INSERT, UPDATE, DELETE) porque la base de datos debe mantenerlo actualizado. No conviene crear indices para columnas que raramente se consultan.

## Estrategia de indices para JPA

Las relaciones `@ManyToOne` y `@OneToOne` son candidatas inmediatas a indice, porque la columna de clave foranea se utiliza en JOINs y busquedas:

```java
@Entity
public class Pedido {
    @ManyToOne
    @JoinColumn(name = "usuario_id")
    private Usuario usuario;
}
```

```sql
CREATE INDEX idx_pedidos_usuario_id ON pedidos (usuario_id);
```

Las columnas utilizadas en `@Query`, query derivation y `ORDER BY` tambien deben estar indexadas. Para busquedas con `Containing` o `Like`, los indices B-tree estandar no son utiles; se necesitan indices de texto completo (GIN, GiST) especificos del motor.

## Consultas lentas comunes en JPA/Hibernate

El patron mas comun de consulta lenta es la falta de indice en la columna de filtro de una query derivation:

```java
List<Pedido> findByUsuarioEmail(String email);
-- Sin indice en usuarios.email, esta consulta ejecuta un Seq Scan en usuarios
```

Otro patron es la ausencia de indice en columnas de ordenamiento con paginacion profunda:

```sql
-- Sin indice en fecha_creacion, cada pagina requiere un Sort completo de la tabla
SELECT * FROM pedidos ORDER BY fecha_creacion DESC LIMIT 20 OFFSET 10000;
```

La solucion es crear el indice correspondiente:

```sql
CREATE INDEX idx_pedidos_fecha ON pedidos (fecha_creacion DESC);
```
