# HikariCP

HikariCP es una libreria de pool de conexiones JDBC escrita en Java. Su nombre proviene del japones "hikari" (luz), en referencia a su rapidez. Es, por amplio margen, el pool de conexiones mas rapido del ecosistema JVM, y desde Spring Boot 2.x es el pool predeterminado, lo que lo convierte en el estandar de facto para aplicaciones Spring.

## Arquitectura y rendimiento

El rendimiento superior de HikariCP se debe a varias decisiones de diseno:
- Uso de `FastList` en lugar de `ArrayList` para evitar la comprobacion de capacidad en las inserciones.
- Implementacion propia de `ConcurrentBag` en lugar de estructuras sincronizadas tradicionales, reduciendo la contenucion de hilos.
- Optimizacion del bytecode mediante la eliminacion de lambdas y metodos inline.
- Gestion eficiente de las conexiones con un algoritmo de prestamo que minimiza el bloqueo.

```text
Benchmark de pools de conexion (conexiones/segundo, mas es mejor):

HikariCP    : 120.000 ops/s
Tomcat CP   :  70.000 ops/s
Vibur       :  40.000 ops/s
DBCP2       :  25.000 ops/s
C3P0        :  12.000 ops/s
```

## Configuracion basica

En Spring Boot, HikariCP se configura automaticamente al incluir `spring-boot-starter-data-jpa` o `spring-boot-starter-jdbc`. No se requiere ninguna dependencia adicional.

```properties
# Configuracion basica de HikariCP en Spring Boot
spring.datasource.url=jdbc:postgresql://localhost:5432/mibd
spring.datasource.username=usuario
spring.datasource.password=contrasena
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000
```

En proyectos sin Spring Boot, la configuracion se realiza directamente sobre `HikariConfig` y `HikariDataSource`:

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mibd");
config.setUsername("usuario");
config.setPassword("contrasena");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);
config.setLeakDetectionThreshold(60000);
config.setPoolName("MiPool");

HikariDataSource ds = new HikariDataSource(config);
```

## Parametros de configuracion

### maximum-pool-size

Define el numero maximo de conexiones que el pool puede mantener, incluyendo las conexiones activas e inactivas. El valor optimo depende de la carga de trabajo y de la capacidad de la base de datos.

```properties
spring.datasource.hikari.maximum-pool-size=20
```

Un valor demasiado alto puede saturar la base de datos. Una regla empirica es `maximum-pool-size = T * (C - 1) + 1`, donde `T` es el numero de hilos y `C` el numero de conexiones por hilo. En la practica, valores entre 10 y 20 son adecuados para la mayoria de aplicaciones.

### minimum-idle

Numero minimo de conexiones inactivas que el pool mantiene abiertas. HikariCP intentara mantener al menos este numero de conexiones disponibles.

```properties
spring.datasource.hikari.minimum-idle=5
```

Si la aplicacion tiene picos de carga repentinos, mantener un `minimum-idle` mayor que 0 reduce la latencia de las primeras peticiones tras un periodo de inactividad.

### connection-timeout

Tiempo maximo en milisegundos que un hilo espera para obtener una conexion del pool antes de lanzar una excepcion `SQLTransientConnectionException`.

```properties
spring.datasource.hikari.connection-timeout=30000
```

Un valor de 0 significa espera infinita, lo cual no es recomendable. Valores tipicos oscilan entre 3000 y 30000 ms.

### idle-timeout

Tiempo maximo en milisegundos que una conexion puede permanecer inactiva en el pool antes de ser cerrada. Solo se aplica cuando el numero de conexiones inactivas supera `minimum-idle`.

```properties
spring.datasource.hikari.idle-timeout=600000
```

El valor por defecto es 600000 (10 minutos). Un valor demasiado bajo causara cierres y aperturas constantes de conexiones.

### max-lifetime

Tiempo maximo de vida de una conexion en milisegundos. Una vez superado, la conexion se cierra aunque este activa. Este parametro evita fugas de recursos por parte de la base de datos o del firewall.

```properties
spring.datasource.hikari.max-lifetime=1800000
```

Debe ser menor que el timeout de conexion configurado en la base de datos. Por ejemplo, si PostgreSQL tiene `tcp_keepalives_idle` en 30 minutos, `max-lifetime` debe ser inferior (1800000 ms = 30 minutos).

### leak-detection-threshold

Activa la deteccion de fugas de conexion. Si una conexion permanece activa mas tiempo del especificado, HikariCP registra un aviso en el log con la traza de la pila donde se obtuvo la conexion.

```properties
spring.datasource.hikari.leak-detection-threshold=60000
```

> Este parametro solo debe usarse en desarrollo. En produccion tiene un coste de rendimiento porque HikariCP debe capturar la traza de la pila en cada `getConnection()`.

```text
WARN  HikariPool-1 - Connection leak detection triggered:
  - Connection com.zaxxer.hikari.ProxyConnection@1234 was returned
    but has not been closed for 60000 ms.
    Stack Trace:
    at com.miapp.repositorio.PedidoRepositorio.buscarPedidos(PedidoRepositorio.java:25)
    at com.miapp.servicio.PedidoServicio.ejecutar(PedidoServicio.java:42)
```

## Validacion de conexiones

HikariCP no ejecuta una consulta de validacion en cada obtencion de conexion. En su lugar, confia en `max-lifetime` para descartar conexiones obsoletas y en `connection-timeout` para manejar fallos momentaneos. No obstante, es posible configurar una validacion:

```properties
spring.datasource.hikari.connection-test-query=SELECT 1
```

> En la mayoria de los casos no es necesario configurar `connection-test-query`. HikariCP valida las conexiones perezosamente al obtenerlas. Solo es recomendable si hay firewalls o proxies que cierren conexiones inactivas de forma silenciosa.
