# Configuracion por base de datos

Cada motor de base de datos tiene caracteristicas particulares que afectan a la configuracion optima del pool de conexiones. Factores como el numero maximo de conexiones admitidas, el tiempo de vida de las conexiones, la latencia de red y el comportamiento ante desconexiones determinan los valores recomendados para cada parametro de HikariCP.

## PostgreSQL

PostgreSQL tiene un modelo de procesos. Cada conexion activa consume un proceso hijo del servidor, por lo que el numero de conexiones impacta directamente en el uso de memoria y CPU del servidor.

```properties
# Configuracion recomendada para PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/mibd
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=5000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.idle-timeout=600000
```

El pool de 20 conexiones es el valor recomendado para la mayoria de aplicaciones. PostgreSQL tiene un limite configurable mediante `max_connections` (tipicamente 100 por defecto). Superar este limite provoca errores de conexion.

El `max-lifetime` de 1800000 ms (30 minutos) permite a PostgreSQL liberar memoria de procesos que mantienen conexiones de larga duracion. El `connection-timeout` de 5000 ms es agresivo pero adecuado para entornos de produccion donde las conexiones deben establecerse rapidamente.

```yaml
# Configuracion YAML para PostgreSQL
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mibd
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 5000
      max-lifetime: 1800000
      idle-timeout: 600000
      pool-name: PoolPostgreSQL
      connection-test-query: SELECT 1
```

PostgreSQL admite `pg_stat_activity` para monitorizar conexiones activas:

```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```

## MySQL

MySQL maneja las conexiones mediante hilos, lo que lo hace menos sensible al numero de conexiones que PostgreSQL. Sin embargo, tiene un timeout de espera (`wait_timeout`) que cierra conexiones inactivas, tipicamente configurado en 28800 segundos (8 horas).

```properties
# Configuracion recomendada para MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/mibd?useSSL=false&serverTimezone=UTC
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=5000
spring.datasource.hikari.max-lifetime=300000
spring.datasource.hikari.idle-timeout=600000
```

El `max-lifetime` de 300000 ms (5 minutos) es significativamente menor que en PostgreSQL. La razon es que los balanceadores de carga y proxies intermedios (como ProxySQL o HAProxy) suelen tener timeouts mas agresivos para MySQL. Un `max-lifetime` corto evita que el pool intente usar conexiones que el intermediario ya ha cerrado.

MySQL tambien tiene un parametro `max_connections` que limita el numero total de conexiones simultaneas. El valor por defecto suele ser 151. Si el pool supera este limite, las nuevas conexiones seran rechazadas.

```properties
# Verificar el limite de conexiones en MySQL
spring.datasource.hikari.connection-init-sql=SET SESSION wait_timeout=300
```

## H2 en testing

Para entornos de testing con H2 en memoria, la configuracion del pool debe adaptarse a un escenario de baja concurrencia y alta rotacion de esquemas.

```properties
# Configuracion recomendada para H2 en testing
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
spring.datasource.hikari.maximum-pool-size=5
spring.datasource.hikari.minimum-idle=1
spring.datasource.hikari.connection-timeout=3000
spring.datasource.hikari.max-lifetime=600000
```

El pool se reduce a 5 conexiones maximas y 1 conexion minima porque los tests rara vez necesitan concurrencia elevada. Valores menores reducen el overhead de creacion de conexiones innecesarias.

El `DB_CLOSE_DELAY=-1` es critico. Sin este parametro, H2 cerraria la base de datos al liberar la ultima conexion, y el pool no podria reutilizarla en la siguiente operacion.

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class RepositorioTest {

    @Autowired
    private DataSource dataSource;

    @Test
    void elPoolDeberiaEstarConfigurado() throws Exception {
        HikariDataSource hikari = (HikariDataSource) dataSource;
        assertThat(hikari.getMaximumPoolSize()).isEqualTo(5);
        assertThat(hikari.getMinimumIdle()).isEqualTo(1);
    }
}
```

## Resumen comparativo

| Parametro | PostgreSQL | MySQL | H2 testing |
|-----------|-----------|-------|------------|
| maximum-pool-size | 20 | 20 | 5 |
| minimum-idle | 5 | 5 | 1 |
| connection-timeout | 5000 ms | 5000 ms | 3000 ms |
| max-lifetime | 1800000 ms | 300000 ms | 600000 ms |
| idle-timeout | 600000 ms | 600000 ms | 600000 ms |
| connection-test-query | SELECT 1 | SELECT 1 | No necesario |

> Los valores de esta tabla son puntos de partida, no configuraciones universales. Cada aplicacion debe ajustar estos parametros en funcion de su carga de trabajo, latencia de red y caracteristicas del servidor de base de datos. La monitorizacion en produccion es la unica forma de determinar la configuracion optima.
