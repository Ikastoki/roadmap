# Pool de conexiones

Establecer una conexion TCP con una base de datos es una operacion costosa: implica handshake de red, autenticacion, negociacion de parametros de sesion y, a menudo, encriptacion TLS. En una aplicacion que atiende cientos de peticiones por segundo, abrir y cerrar una conexion para cada operacion de base de datos destruiria el rendimiento. Un pool de conexiones mantiene un conjunto de conexiones abiertas y reutilizables, eliminando la sobrecarga de creacion y destruccion en cada operacion.

Cuando una operacion necesita acceso a la base de datos, solicita una conexion del pool. Si hay una disponible, la recibe inmediatamente. Si no, la solicitud queda en espera hasta que otra operacion libere una conexion o hasta que se cumpla el tiempo de espera maximo configurado. Al finalizar la operacion, la conexion no se cierra: se devuelve al pool para ser reutilizada.

## HikariCP

HikariCP es la implementacion de pool de conexiones mas utilizada en el ecosistema Java, y la recomendada por Spring Boot desde la version 2.0. Su nombre proviene del japones "hikari" (luz), reflejando su objetivo de ser extremadamente rapido y liviano. HikariCP esta disenhado con un enfoque en la eficiencia: su tamaño de codigo es reducido, utiliza estructuras de datos optimizadas para concurrencia (como `ConcurrentBag`) y evita bloqueos innecesarios mediante tecnicas como el spin-lock y el uso de `CAS` (Compare-And-Swap) en lugar de sincronizacion pesada.

Las caracteristicas que distinguen a HikariCP de otras implementaciones (como Apache DBCP2 o C3P0) son:

- **Overhead minimo**: el tiempo de tomar y devolver una conexion del pool se mide en microsegundos.
- **Sin bloqueos en el camino critico**: las operaciones de `getConnection()` y `close()` no realizan llamadas de sincronizacion costosas.
- **Gestion proactiva de conexiones muertas**: verifica la validez de las conexiones antes de entregarlas, detectando cortes de red o reinicios de base de datos.
- **Metricas integradas**: expone contadores de conexiones activas, inactivas, pendientes y tiempo de adquisicion.

## Configuracion

HikariCP se configura mediante propiedades que controlan el tamaño del pool, los tiempos de espera y la politica de validacion de conexiones. En Spring Boot, la configuracion tipica va en `application.properties` o `application.yml`.

```properties
# Configuracion de HikariCP en Spring Boot
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-test-query=SELECT 1
```

En Java puro, sin Spring Boot, la configuracion se realiza directamente sobre `HikariConfig`:

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/miapp");
config.setUsername("usuario");
config.setPassword("password");
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);
config.setConnectionTestQuery("SELECT 1");

HikariDataSource dataSource = new HikariDataSource(config);
```

### Parametros esenciales

**maximumPoolSize**: numero maximo de conexiones que el pool puede contener, incluyendo las activas y las inactivas. Es el parametro mas importante y el que mas impacto tiene en el rendimiento. Un valor demasiado bajo provoca contention (las peticiones esperan por una conexion); un valor demasiado alto satura la base de datos con conexiones ociosas que consumen memoria y recursos del servidor.

La regla general propuesta por los autores de HikariCP es:

```
maximumPoolSize = Tn * (Cm - 1) + 1
```

Donde `Tn` es el numero de hilos de ejecucion simultaneos en la aplicacion y `Cm` es el numero de conexiones que cada tarea necesita. En la practica, para la mayoria de aplicaciones web con un pool de hilos de 200, un `maximumPoolSize` de 10-15 es suficiente porque las conexiones se liberan rapidamente. Valores superiores a 20-30 rara vez mejoran el rendimiento y suelen empeorarlo.

> El mito de "mas conexiones = mas rendimiento" es falso. Las bases de datos son sistemas concurrentes con recursos limitados (CPU, memoria, locks de buffer pool). Cada conexion adicional consume memoria y compite por los mismos recursos. Mas alla de un punto optimo, anadir conexiones reduce el rendimiento por aumento de contention interna en la base de datos.

**minimumIdle**: numero minimo de conexiones inactivas que el pool mantiene. Si cae por debajo de este valor, HikariCP crea nuevas conexiones proactivamente. Un valor bajo reduce el consumo de recursos en periodos de baja actividad; un valor alto elimina la latencia de creacion cuando llega un pico de peticiones. Tipicamente se configura entre 2 y la mitad de `maximumPoolSize`.

**connectionTimeout**: tiempo maximo en milisegundos que una peticion espera por una conexion del pool. Si se agota el tiempo, se lanza una excepcion `SQLTransientConnectionException`. El valor por defecto es 30000 ms (30 segundos). Reducirlo permite detectar rapidamente problemas de contention; aumentarlo evita falsos positivos en picos transitorios.

**idleTimeout**: tiempo en milisegundos que una conexion inactiva permanece en el pool antes de ser cerrada. Solo se aplica cuando el numero de conexiones inactivas supera `minimumIdle`. El valor por defecto es 600000 ms (10 minutos). No debe ser inferior a `connectionTimeout` ni superior a `maxLifetime`.

**maxLifetime**: tiempo maximo de vida de una conexion en milisegundos. Independientemente de la actividad, la conexion se cierra al cumplir esta edad. Es necesario porque las conexiones TCP/IP pueden ser interrumpidas por firewalls, balanceadores de carga o la propia base de datos sin que el pool lo detecte. El valor recomendado es 1800000 ms (30 minutos), ligeramente inferior al tiempo de espera de cierre de conexiones inactivas de la mayoria de bases de datos.

### Gestion de conexiones muertas

Las conexiones pueden volverse inutilizables sin previo aviso: un reinicio de la base de datos, un timeout de firewall, una caida de red. HikariCP ofrece mecanismos para detectar y descartar estas conexiones:

- **connectionTestQuery**: sentencia SQL que se ejecuta para verificar la validez de una conexion antes de entregarla. En PostgreSQL o MySQL, `SELECT 1` es suficiente. En algunos drivers (Oracle), se necesita `SELECT 1 FROM DUAL`.
- **HikariCP automatico**: desde la version actual, HikariCP puede determinar automaticamente si el driver JDBC soporta el metodo `isValid()` (disponible en JDBC 4+). Si es asi, no es necesario configurar `connectionTestQuery`.

Ademas, HikariCP ejecuta un thread de mantenimiento periodico que recorre las conexiones inactivas y las valida, cerrando las que fallan y creando nuevas para mantener el `minimumIdle`.

> Configurar un `connectionTestQuery` demasiado pesado (como `SELECT * FROM una_tabla_grande`) contrarresta el beneficio del pool al anadir latencia a cada operacion de obtencion de conexion. Usar `SELECT 1` o confiar en `isValid()` del driver es suficiente.

## Monitoreo del pool

HikariCP expone metricas a traves de la interfaz `HikariDataSourceMXBean` (JMX) y, en Spring Boot, a traves de los indicadores de salud de Actuator. Las metricas mas relevantes son:

- **ActiveConnections**: conexiones actualmente en uso.
- **IdleConnections**: conexiones disponibles en el pool.
- **PendingThreads**: hilos esperando por una conexion.
- **TotalConnections**: suma de activas e inactivas.
- **ConnectionTimeoutRate**: proporcion de peticiones que agotaron el timeout.

Un valor alto de `PendingThreads` o un `ConnectionTimeoutRate` positivo indica que el `maximumPoolSize` es insuficiente o que las conexiones se retienen durante demasiado tiempo (una transaccion lenta, un cierre de conexion olvidado, un fetch masivo sin paginacion).
