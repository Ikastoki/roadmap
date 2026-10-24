# Monitoreo del pool

La capacidad de observar el estado interno del pool de conexiones es fundamental para diagnosticar problemas de rendimiento, detectar fugas de conexion y dimensionar correctamente los recursos de la aplicacion. HikariCP expone metricas detalladas a traves de JMX y de su integracion con Spring Boot Actuator.

## HikariPoolMXBean

HikariCP registra un bean JMX llamado `HikariPoolMXBean` que permite inspeccionar el estado del pool en tiempo real. Se accede a el mediante cualquier herramienta compatible con JMX (JConsole, VisualVM, o programaticamente).

```java
HikariDataSource ds = new HikariDataSource(config);
HikariPoolMXBean mxBean = ds.getHikariPoolMXBean();

int activas = mxBean.getActiveConnections();       // Conexiones en uso
int inactivas = mxBean.getIdleConnections();        // Conexiones disponibles
int esperando = mxBean.getPendingThreads();         // Hilos esperando conexion
int totales = mxBean.getTotalConnections();         // Total de conexiones creadas
```

### Metricas clave

- **ActiveConnections**: numero de conexiones actualmente en uso por la aplicacion. Si este valor se acerca a `maximum-pool-size` de forma sostenida, el pool esta saturado y puede haber hilos esperando.
- **IdleConnections**: conexiones disponibles para ser usadas. Si es 0 y hay peticiones entrantes, los hilos entraran en espera hasta que una conexion se libere o hasta que venza el `connection-timeout`.
- **PendingThreads**: hilos bloqueados esperando una conexion disponible. En condiciones normales debe ser 0. Un valor distinto de 0 indica que el pool esta bajo presion.
- **TotalConnections**: numero total de conexiones gestionadas por el pool. Es la suma de activas e inactivas.

```sql
-- Consulta equivalente en PostgreSQL para comparar
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
```

## Leak detection

HikariCP incluye un mecanismo de deteccion de fugas de conexion. Cuando se activa `leak-detection-threshold`, el pool registra un aviso si una conexion permanece activa mas tiempo del configurado.

```properties
spring.datasource.hikari.leak-detection-threshold=60000
```

```text
WARN  HikariPool-1 - Connection leak detection triggered:
  - Connection com.zaxxer.hikari.ProxyConnection@1234 was returned
    but has not been closed for 60000 ms.
    Stack Trace:
    at com.miapp.repositorio.PedidoRepositorio.buscarPedidos(PedidoRepositorio.java:25)
```

La traza de la pila muestra exactamente donde se obtuvo la conexion que no se cerro, lo que permite localizar el problema sin necesidad de revisar todo el codigo.

> El leak detection tiene un coste de rendimiento porque HikariCP debe capturar la traza de la pila en cada `getConnection()`. En produccion se recomienda desactivarlo o usar un valor alto unicamente durante periodos de diagnostico.

## Slow query detection

HikariCP no incluye deteccion de consultas lentas de forma nativa. Sin embargo, combinado con las metricas de conexion, es posible inferir problemas de rendimiento: si `activeConnections` se mantiene alto y `pendingThreads` crece, es probable que haya consultas lentas bloqueando las conexiones.

Para una deteccion mas precisa, se recomienda configurar `spring.jpa.properties.hibernate.session.events.log` o utilizar un proxy JDBC como `datasource-proxy` o `p6spy`.

```properties
# Registro de consultas lentas con datasource-proxy
spring.datasource.hikari.leak-detection-threshold=5000
```

```java
// Envolver el DataSource para loggear consultas lentas
ProxyDataSource proxy = new ProxyDataSourceBuilder(hikariDs)
    .logSlowQueryBySlf4j(3000, TimeUnit.MILLISECONDS)
    .build();
```

## Spring Boot Actuator

Spring Boot Actuator expone las metricas de HikariCP a traves del endpoint `/actuator/metrics`. Para habilitarlo:

```properties
management.endpoints.web.exposure.include=metrics
management.metrics.export.prometheus.enabled=true
```

Las metricas disponibles son:
- `hikaricp.connections.active`: conexiones activas.
- `hikaricp.connections.idle`: conexiones inactivas.
- `hikaricp.connections.pending`: hilos esperando conexion.
- `hikaricp.connections.max`: tamano maximo del pool.
- `hikaricp.connections.min`: tamano minimo del pool.
- `hikaricp.connections.total`: total de conexiones.
- `hikaricp.connections.timeout`: numero de veces que se ha superado el connection-timeout.
- `hikaricp.connections.creation.seconds`: tiempo de creacion de nuevas conexiones.

```bash
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
```

```json
{
  "name": "hikaricp.connections.active",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 3
    }
  ],
  "availableTags": [
    {"tag": "pool", "values": ["HikariPool-1"]}
  ]
}
```

Para una integracion con Prometheus y Grafana, las metricas se exportan automaticamente si se incluye `micrometer-registry-prometheus`:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

> En produccion, se recomienda configurar alertas basadas en estas metricas. Por ejemplo, si `hikaricp.connections.pending` supera 0 durante mas de 5 minutos, es indicio de que el pool esta infradimensionado o de que hay consultas lentas bloqueando conexiones.

## Health check con Actuator

El endpoint `/actuator/health` muestra el estado del pool HikariCP como parte del health check de la base de datos:

```properties
management.endpoint.health.show-details=always
```

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "SELECT 1"
      }
    }
  }
}
```

Si el pool no puede obtener una conexion de validacion, el estado cambia a `DOWN`, lo que permite a orquestadores como Kubernetes reiniciar el pod automaticamente.
