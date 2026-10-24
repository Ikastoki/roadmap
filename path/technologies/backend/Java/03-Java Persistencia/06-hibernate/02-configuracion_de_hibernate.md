# Configuracion de Hibernate

Hibernate se configura tradicionalmente mediante el archivo `hibernate.cfg.xml`, aunque soporta configuracion programatica a traves de `Configuration` y tambien la configuracion automatica que ofrece Spring Boot mediante `application.properties`. Las tres aproximaciones son validas y la eleccion depende del tipo de proyecto y del grado de control que se necesite.

## Configuracion con hibernate.cfg.xml

El archivo `hibernate.cfg.xml` se coloca en el classpath (generalmente en `src/main/resources`) y es el punto de entrada clasico para cualquier aplicacion que use Hibernate de forma directa, sin Spring.

```xml
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <!-- Conexion a la base de datos -->
        <property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
        <property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/mibd</property>
        <property name="hibernate.connection.username">usuario</property>
        <property name="hibernate.connection.password">password</property>

        <!-- Dialecto SQL -->
        <property name="hibernate.dialect">org.hibernate.dialect.PostgreSQLDialect</property>

        <!-- Mostrar SQL generado en consola -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>

        <!-- Estrategia DDL -->
        <property name="hibernate.hbm2ddl.auto">update</property>

        <!-- Pool de conexiones (C3P0) -->
        <property name="hibernate.c3p0.min_size">5</property>
        <property name="hibernate.c3p0.max_size">20</property>
        <property name="hibernate.c3p0.timeout">300</property>
        <property name="hibernate.c3p0.max_statements">50</property>

        <!-- Cache de segundo nivel -->
        <property name="hibernate.cache.use_second_level_cache">true</property>
        <property name="hibernate.cache.region.factory_class">
            org.hibernate.cache.ehcache.EhCacheRegionFactory
        </property>

        <!-- Cache de consultas -->
        <property name="hibernate.cache.use_query_cache">true</property>

        <!-- Mapeo de entidades -->
        <mapping class="com.ejemplo.modelo.Usuario"/>
        <mapping class="com.ejemplo.modelo.Pedido"/>
    </session-factory>
</hibernate-configuration>
```

La propiedad `hibernate.dialect` es fundamental: le indica a Hibernate el dialecto SQL que debe usar para generar sentencias compatibles con el motor concreto. Cada base de datos tiene su propio dialecto (`PostgreSQLDialect`, `MySQLDialect`, `OracleDialect`, `H2Dialect`, etc.) y elegir el incorrecto puede producir SQL invalido o comportamientos inesperados.

La propiedad `hbm2ddl.auto` controla como Hibernate gestiona el esquema de la base de datos:

- **create**: Borra el esquema existente y lo crea de nuevo cada vez que se arranca la aplicacion. Util en desarrollo pero peligroso en produccion.
- **update**: Compara el mapeo de entidades con el esquema existente e intenta modificarlo sin borrar datos. No garantiza la integridad en cambios complejos.
- **validate**: Solo comprueba que el esquema existente coincida con el mapeo. No realiza cambios.
- **none** (o ausencia de propiedad): No realiza ninguna accion sobre el esquema.

> `update` es util en desarrollo pero no debe usarse en produccion. En entornos productivos se recomienda `validate` combinado con herramientas de migracion como Flyway o Liquibase.

## Configuracion programatica

Cuando se necesita construir la configuracion en tiempo de ejecucion, por ejemplo para entornos con multiples bases de datos o configuracion dinamica, se utiliza la clase `Configuration`:

```java
Configuration configuration = new Configuration()
    .setProperty("hibernate.connection.driver_class", "org.postgresql.Driver")
    .setProperty("hibernate.connection.url", "jdbc:postgresql://localhost:5432/mibd")
    .setProperty("hibernate.connection.username", "usuario")
    .setProperty("hibernate.connection.password", "password")
    .setProperty("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect")
    .setProperty("hibernate.hbm2ddl.auto", "validate")
    .setProperty("hibernate.show_sql", "true")
    .addAnnotatedClass(Usuario.class)
    .addAnnotatedClass(Pedido.class);

StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .applySettings(configuration.getProperties())
    .build();

SessionFactory sessionFactory = configuration.buildSessionFactory(registry);
```

La configuracion programatica es especialmente util en tests unitarios donde se necesita levantar Hibernate con H2 en memoria:

```java
Configuration config = new Configuration()
    .setProperty("hibernate.connection.driver_class", "org.h2.Driver")
    .setProperty("hibernate.connection.url", "jdbc:h2:mem:testdb")
    .setProperty("hibernate.dialect", "org.hibernate.dialect.H2Dialect")
    .setProperty("hibernate.hbm2ddl.auto", "create-drop");
```

## Configuracion en Spring Boot

Spring Boot simplifica radicalmente la configuracion. Con solo incluir `spring-boot-starter-data-jpa` y la dependencia del driver de base de datos, las propiedades se definen en `application.properties`:

```properties
# DataSource
spring.datasource.url=jdbc:postgresql://localhost:5432/mibd
spring.datasource.username=usuario
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Pool de conexiones (HikariCP es el default en Spring Boot 2+)
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.max-lifetime=1800000

# Cache de segundo nivel
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory
```

> En Spring Boot, `spring.jpa.hibernate.ddl-auto` es el equivalente directo de `hibernate.hbm2ddl.auto`. Spring Boot configura automaticamente el `EntityManagerFactory` y el `TransactionManager`, por lo que no es necesario crear manualmente la `SessionFactory`.

## Pool de conexiones

Hibernate incluye soporte integrado para C3P0 y HikariCP. Desde Hibernate 5.4 en adelante, HikariCP es el pool recomendado y se configura mediante las propiedades `hibernate.hikari.*`. En Spring Boot, HikariCP es el pool por defecto y se configura a traves de las propiedades `spring.datasource.hikari.*`.

```properties
# Pool con HikariCP via propiedades nativas de Hibernate
spring.jpa.properties.hibernate.hikari.minimumIdle=5
spring.jpa.properties.hibernate.hikari.maximumPoolSize=20
spring.jpa.properties.hibernate.hikari.idleTimeout=30000
```
