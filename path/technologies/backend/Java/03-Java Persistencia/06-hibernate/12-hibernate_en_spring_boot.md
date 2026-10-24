# Hibernate en Spring Boot

Spring Boot integra Hibernate de forma automatica a traves del modulo `spring-boot-starter-data-jpa`. Cuando se incluye esta dependencia, Spring Boot configura automaticamente el `DataSource`, el `EntityManagerFactory`, el `JpaTransactionManager` y las propiedades de Hibernate sin necesidad de archivos XML ni clases de configuracion adicionales.

## Configuracion automatica

Spring Boot realiza una serie de autoconfiguraciones al detectar JPA y Hibernate en el classpath:

1. Crea un `DataSource` a partir de las propiedades `spring.datasource.*`. Si no se configura ninguna base de datos explicita y H2 esta en el classpath, Spring Boot levanta una base de datos H2 en memoria por defecto.
2. Configura el `EntityManagerFactory` de JPA usando `HibernateJpaVendorAdapter`, que establece automaticamente el dialecto SQL, el ddl-auto y otras propiedades de Hibernate.
3. Escanea las entidades anotadas con `@Entity` en el paquete donde esta la clase principal de la aplicacion y en sus subpaquetes.
4. Expone `EntityManager` como bean que puede inyectarse directamente con `@PersistenceContext`.

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@Service
public class UsuarioService {

    @PersistenceContext
    private EntityManager entityManager;

    public Usuario buscarPorEmail(String email) {
        return entityManager.createQuery(
            "FROM Usuario u WHERE u.email = :email", Usuario.class)
            .setParameter("email", email)
            .uniqueResult();
    }
}
```

## application.properties

Las propiedades de configuracion para JPA y Hibernate se definen en `application.properties` o `application.yml` bajo los prefijos `spring.jpa` y `spring.jpa.properties`.

```properties
# DataSource
spring.datasource.url=jdbc:postgresql://localhost:5432/mibd
spring.datasource.username=usuario
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# Configuracion JPA general
spring.jpa.open-in-view=false
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.highlight_sql=true

# DDL
spring.jpa.hibernate.ddl-auto=validate

# Dialecto (Spring Boot lo detecta automaticamente, pero se puede forzar)
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Batching
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

# Cache de segundo nivel
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory

# Pool HikariCP
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.connection-timeout=5000
spring.datasource.hikari.max-lifetime=1800000
```

### Propiedades clave

- **spring.jpa.open-in-view**: Cuando es `true` (valor por defecto), mantiene el EntityManager abierto durante toda la peticion HTTP para permitir la carga perezosa de colecciones en la vista. Esto puede provocar problemas de rendimiento y errores de conexion si no se gestiona adecuadamente. Se recomienda establecerlo a `false` si no se necesita.

> `open-in-view=true` es una configuracion comoda pero peligrosa. Mantiene la sesion de Hibernate abierta durante toda la peticion, lo que significa que cualquier acceso a una coleccion lazy en la vista disparara una consulta SQL adicional. Ademas, si la vista tarda en renderizarse, la conexion a la base de datos permanece ocupada.

- **spring.jpa.show-sql**: Muestra las sentencias SQL generadas por Hibernate en la salida estandar. Combina con `spring.jpa.properties.hibernate.format_sql=true` para una salida legible.

- **spring.jpa.hibernate.ddl-auto**: Controla la generacion del esquema. Los valores son los mismos que `hibernate.hbm2ddl.auto`: `create`, `create-drop`, `update`, `validate`, `none`. El valor por defecto en Spring Boot es `create-drop` si se usa una base de datos embebida (H2, HSQL, Derby) y `none` en caso contrario.

## Configuracion avanzada

### Multiples fuentes de datos

Spring Boot permite configurar varios `DataSource` y `EntityManagerFactory` para aplicaciones que necesitan acceder a multiples bases de datos:

```java
@Configuration
public class DatasourceConfig {

    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return builder
                .dataSource(dataSource)
                .packages("com.ejemplo.modelo.primary")
                .persistenceUnit("primary")
                .build();
    }

    @Bean(name = "secondaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean secondaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("secondaryDataSource") DataSource dataSource) {
        return builder
                .dataSource(dataSource)
                .packages("com.ejemplo.modelo.secondary")
                .persistenceUnit("secondary")
                .build();
    }
}
```

### Personalizacion con @Configuration

Si se necesita un control total sobre la configuracion de Hibernate, se puede crear un bean `JpaProperties` o `HibernateProperties` personalizado:

```java
@Configuration
public class HibernateConfig {

    @Bean
    public HibernatePropertiesCustomizer hibernateCustomizer() {
        return props -> {
            props.put("hibernate.jdbc.batch_size", 100);
            props.put("hibernate.order_inserts", true);
            props.put("hibernate.order_updates", true);
            props.put("hibernate.jdbc.fetch_size", 200);
        };
    }
}
```

### Uso con Spring Data JPA

Aunque Spring Data JPA abstrae la mayoria de las operaciones de persistencia mediante repositorios, el conocimiento de Hibernate sigue siendo esencial para configurar el rendimiento, el cache y el manejo de transacciones:

```java
@Repository
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {

    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    Optional<Usuario> findByEmail(String email);

    @Modifying
    @Query("UPDATE Usuario u SET u.activo = false WHERE u.ultimoAcceso < :fecha")
    int desactivarInactivos(@Param("fecha") LocalDateTime fecha);
}
```

Los hints de Hibernate y las consultas nativas estan disponibles incluso cuando se usa Spring Data JPA, lo que permite combinar la comodidad de los repositorios con el control fino de Hibernate cuando es necesario.
