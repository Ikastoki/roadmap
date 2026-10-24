# Connection pool tuning

El pool de conexiones es el componente que gestiona las conexiones a la base de datos, reutilizandolas en lugar de abrir y cerrar una conexion por cada operacion. Una configuracion inadecuada del pool puede provocar desde tiempos de respuesta lentos hasta fallos completos del sistema por agotamiento de conexiones. HikariCP es el pool de conexiones por defecto en Spring Boot 2.x y 3.x, reconocido por su rendimiento y baja latencia.

## Tamaño del pool

La formula clasica para calcular el tamaño optimo del pool es una guia de partida, no una regla absoluta:

```
Pool size = CPU_cores * 2 + disk_spindles
```

En la practica, para aplicaciones modernas con almacenamiento SSD, la formula se simplifica:

```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
```

`maximum-pool-size` define el numero maximo de conexiones que el pool puede mantener abiertas simultaneamente. Un valor demasiado bajo provoca que las peticiones esperen por una conexion disponible. Un valor demasiado alto satura la base de datos con conexiones inactivas que compiten por recursos (CPU, memoria, locks).

El punto de partida recomendado para aplicaciones tipicas es:

- Aplicacion con 2-4 instancias, base de datos dedicada: pool de 10-20 por instancia
- Microservicio con baja concurrencia: pool de 5-10
- Aplicacion con alta concurrencia y queries rapidas: pool de 20-30

> El tamaño optimo del pool no es "cuanto mas grande, mejor". Cada conexion consume aproximadamente 1MB de RAM en el servidor de base de datos, y un pool sobredimensionado aumenta la contencion por locks en la base de datos.

## Tiempo de espera y timeout

```properties
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

`connection-timeout` es el tiempo maximo (en milisegundos) que una peticion espera por una conexion del pool antes de lanzar una excepcion. 30 segundos es un valor razonable para la mayoria de aplicaciones.

`idle-timeout` es el tiempo maximo que una conexion permanece inactiva en el pool antes de ser cerrada. Por defecto son 10 minutos.

`max-lifetime` es la vida maxima de una conexion, independientemente de su actividad. Debe ser ligeramente inferior al timeout de conexion del lado de la base de datos. Por ejemplo, si MySQL tiene `wait_timeout=28800` (8 horas), `max-lifetime` debe ser 1800000 (30 minutos) o menos para evitar que el pool intente usar conexiones cerradas por la base de datos.

## Deteccion de leaks

Las fugas de conexion (connection leaks) ocurren cuando el codigo obtiene una conexion del pool pero no la libera. HikariCP proporciona mecanismos para detectarlas:

```properties
spring.datasource.hikari.leak-detection-threshold=60000
```

Si una conexion permanece ocupada durante mas de 60 segundos, HikariCP registra un warning en el log con el stack trace del punto donde se obtuvo la conexion. Esto permite identificar rapidamente el codigo responsable de la fuga.

En desarrollo, se puede reducir el umbral a 5-10 segundos para detectar fugas temprano. En produccion, un valor de 30-60 segundos evita falsos positivos por consultas legitimas de larga duracion.

## Connection validation

HikariCP valida las conexiones antes de entregarlas para asegurar que siguen siendo funcionales. La configuracion por defecto es suficiente para la mayoria de los casos:

```properties
spring.datasource.hikari.connection-test-query=SELECT 1
```

`connection-test-query` es una consulta ligera que HikariCP ejecuta para verificar que la conexion es valida. Cada motor de base de datos tiene su propia consulta de prueba:

- PostgreSQL: `SELECT 1`
- MySQL/MariaDB: `SELECT 1`
- H2: `SELECT 1`
- Oracle: `SELECT 1 FROM DUAL`
- MSSQL: `SELECT 1`

Si no se especifica, HikariCP utiliza `isValid()` de JDBC 4, que es mas eficiente porque no ejecuta una consulta real sino que verifica el estado interno de la conexion.

```properties
# Usar isValid() de JDBC (mas rapido, no requiere consulta)
# No establecer connection-test-query
```

## Monitoring del pool

HikariCP expone metricas a traves de Micrometer y Actuator para monitorizar el estado del pool en tiempo real:

```properties
management.endpoints.web.exposure.include=health,metrics
management.metrics.export.prometheus.enabled=true
```

Metricas clave a monitorizar:

- `hikaricp.connections.active`: conexiones actualmente en uso
- `hikaricp.connections.idle`: conexiones inactivas disponibles
- `hikaricp.connections.pending`: peticiones esperando por una conexion
- `hikaricp.connections.timeout`: conexiones que han excedido el timeout
- `hikaricp.connections.max`: tamaño maximo configurado

Un numero persistente de `pending` mayor que cero indica que el pool es demasiado pequeno para la carga actual. Un numero de `active` consistentemente cercano a `max` sugiere que el pool necesita ampliarse o que las consultas son demasiado lentas.

```properties
spring.datasource.hikari.pool-name=MyPool
```

Asignar un nombre al pool facilita la identificacion en logs y metricas cuando hay multiples fuentes de datos.

## Configuracion completa recomendada

```properties
# HikariCP configuration
spring.datasource.hikari.pool-name=AppPool
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000
spring.datasource.hikari.connection-test-query=SELECT 1
spring.datasource.hikari.auto-commit=false
```

`auto-commit=false` deshabilita el auto-commit de JDBC, delegando el control transaccional a Spring. Es recomendable para aplicaciones que gestionan transacciones explicitamente.
