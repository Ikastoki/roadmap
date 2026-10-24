# Pool de conexiones

Establecer una conexion JDBC es una operacion costosa. El proceso implica establecer una conexion TCP, autenticar contra el servidor, negociar parametros de sesion (zona horaria, codificacion, nivel de aislamiento) y, en algunos motores, ejecutar consultas de inicializacion. Todo este proceso puede consumir entre 10 y 100 milisegundos por conexion. En aplicaciones con alta concurrencia, crear y destruir conexiones por cada peticion resulta inaceptable.

Un pool de conexiones (connection pool) mantiene un conjunto de conexiones previamente creadas y las reutiliza entre distintas peticiones. Cuando una operacion necesita acceso a la base de datos, solicita una conexion del pool; cuando termina, la devuelve al pool en lugar de cerrarla fisicamente. Esto reduce drasticamente la latencia de las operaciones y limita el numero maximo de conexiones simultaneas contra el servidor.

## Problemas de crear conexiones manualmente

**Latencia.** Cada nueva conexion requiere handshake TCP y autenticacion. En aplicaciones web con cientos de peticiones por segundo, el tiempo dedicado a crear conexiones puede suponer el 30-50% del tiempo total de respuesta.

**Limite de conexiones del servidor.** Los motores de base de datos tienen un numero maximo de conexiones simultaneas (PostgreSQL por defecto permite 100, MySQL 151). Si cada peticion abre una conexion y no la cierra rapidamente, se agota el limite y las nuevas peticiones quedan en cola o reciben error.

**Fugas de conexiones.** Si una conexion no se cierra correctamente (por ejemplo, por una excepcion no capturada), se pierde permanentemente hasta que expire el timeout del servidor. Con pool, las conexiones "fugas" se detectan y se cierran o reemplazan.

## HikariCP

HikariCP (Hikari significa "luz" en japones) es el pool de conexiones mas utilizado en el ecosistema Java actual. Es el pool por defecto en Spring Boot 2.x y 3.x, y ofrece el mejor rendimiento en las pruebas comparativas. Sus caracteristicas principales:

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

**Configuracion basica:**

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mi_bd");
config.setUsername("postgres");
config.setPassword("postgres");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);

HikariDataSource dataSource = new HikariDataSource(config);
```

O mediante fichero de propiedades:

```java
HikariConfig config = new HikariConfig("/ruta/hikari.properties");
HikariDataSource dataSource = new HikariDataSource(config);
```

### Parametros clave

**maximumPoolSize.** Numero maximo de conexiones que el pool mantiene simultaneamente, incluyendo las conexiones inactivas (idle) y las activas (en uso). El valor optimo depende del numero de peticiones concurrentes, del tiempo que cada peticion retiene la conexion y del limite maximo del servidor de base de datos. Una formula orientativa: `maximumPoolSize = (num_hilos_concurrentes * tiempo_medio_consulta) / tiempo_total_peticion`. Valores tipicos: 10-50.

> No debe confundirse con el numero de conexiones que la aplicacion necesita: si cada peticion retiene la conexion 50 ms y el servidor puede servir 100 peticiones simultaneas, el pool necesitara aproximadamente 5-10 conexiones (100 peticiones/segundo * 0.05 segundos = 5 conexiones simultaneas). Mas conexiones no siempre es mejor; cada conexion consume memoria en el servidor (tipicamente 5-10 MB por conexion).

**minimumIdle.** Numero minimo de conexiones inactivas que el pool mantiene abiertas. Si el pool baja de este umbral, crea nuevas conexiones incluso sin peticiones activas. Valores tipicos: 5-10.

**connectionTimeout.** Tiempo maximo (en milisegundos) que una peticion espera por una conexion del pool si todas estan ocupadas. Si se alcanza el timeout, se lanza `SQLException`. Valor por defecto: 30000 (30 segundos).

**idleTimeout.** Tiempo maximo (en milisegundos) que una conexion inactiva permanece en el pool antes de cerrarse. Solo se aplica si el pool tiene mas conexiones que `minimumIdle`. Valor por defecto: 600000 (10 minutos).

**maxLifetime.** Tiempo maximo de vida de una conexion en el pool, independientemente de su actividad. Al alcanzar este limite, la conexion se cierra y se reemplaza por una nueva. Es recomendable establecerlo ligeramente por debajo del timeout de conexion del servidor de base de datos para evitar que el servidor cierre conexiones que el pool considera activas. Valor por defecto: 1800000 (30 minutos).

**validationTimeout.** Tiempo maximo para validar que una conexion sigue activa (mediante una consulta `SELECT 1`). Valor por defecto: 5000 (5 segundos).

**leakDetectionThreshold.** Tiempo (en milisegundos) a partir del cual una conexion se considera "fugada". Si una conexion se mantiene ocupada mas tiempo del especificado, HikariCP registra un warning con el stack trace del punto donde se obtuvo la conexion. Util para depurar codigo que no devuelve las conexiones al pool. Valor por defecto: 0 (desactivado).

## Apache DBCP2

Apache Commons DBCP2 es el pool de conexiones del proyecto Commons de Apache. Fue muy popular antes de la aparicion de HikariCP, pero su rendimiento es inferior y su configuracion mas compleja.

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-dbcp2</artifactId>
    <version>2.12.0</version>
</dependency>
```

```java
BasicDataSource dataSource = new BasicDataSource();
dataSource.setUrl("jdbc:postgresql://localhost:5432/mi_bd");
dataSource.setUsername("postgres");
dataSource.setPassword("postgres");
dataSource.setMaxTotal(20);
dataSource.setMaxIdle(5);
dataSource.setMinIdle(2);
dataSource.setMaxWaitMillis(30000);
```

## C3P0

C3P0 fue otro pool historico en el ecosistema Java, especialmente popular en la era de Hibernate 3. Actualmente esta en desuso y no se recomienda para nuevas aplicaciones.

```xml
<dependency>
    <groupId>com.mchange</groupId>
    <artifactId>c3p0</artifactId>
    <version>0.9.5.5</version>
</dependency>
```

```java
ComboPooledDataSource dataSource = new ComboPooledDataSource();
dataSource.setJdbcUrl("jdbc:postgresql://localhost:5432/mi_bd");
dataSource.setUser("postgres");
dataSource.setPassword("postgres");
dataSource.setMaxPoolSize(20);
dataSource.setMinPoolSize(5);
dataSource.setAcquireIncrement(3);
```

## Comparativa

| Caracteristica | HikariCP | DBCP2 | C3P0 |
|---------------|----------|-------|------|
| Rendimiento | Excelente | Bueno | Medio |
| Consumo memoria | Bajo (~100KB) | Medio (~500KB) | Alto (~2MB) |
| Facilidad configuracion | Muy alta | Media | Media |
| Soporte JMX | Si | Si | Si |
| Validacion conexiones | Automatica | Configurable | Configurable |
| Deteccion fugas | Si | Si | Si |
| Estado actual | Activo, recomendado | Mantenimiento | Desuso |

HikariCP es la opcion recomendada para cualquier proyecto nuevo por su rendimiento superior, su bajo consumo de memoria y su integracion directa con Spring Boot. DBCP2 puede considerarse en entornos con restricciones de licencia o requisitos especificos de la familia Apache. C3P0 no debe usarse en proyectos nuevos.

> Configurar correctamente el pool de conexiones es una de las optimizaciones de rendimiento mas impactantes que pueden realizarse en una aplicacion Java. Un pool sobredimensionado satura el servidor de base de datos. Un pool infradimensionado provoca colas y timeouts. La monitorizacion del pool mediante JMX (HikariCP expone beans JMX con metricas de conexiones activas, pendientes y en espera) permite ajustar los parametros con datos reales de produccion.
