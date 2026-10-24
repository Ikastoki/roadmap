# Configuracion de JPA

JPA ofrece dos vias principales para configurar la capa de persistencia: el archivo descriptivo `persistence.xml`, que es el mecanismo estandar y obligatorio en Jakarta EE, y la configuracion programatica mediante propiedades, que Spring Boot simplifica hasta casi hacer desaparecer. Ambas opciones producen el mismo resultado: un `EntityManagerFactory` listo para crear `EntityManager`.

## persistence.xml

El archivo `persistence.xml` se coloca en `META-INF/persistence.xml` (dentro del classpath) y define las unidades de persistencia. Una unidad de persistencia agrupa un conjunto de entidades gestionadas por el mismo `EntityManagerFactory`, con una configuracion comun de conexion, dialecto y comportamiento.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
                                 https://jakarta.ee/xml/ns/persistence/persistence_3_0.xsd"
             version="3.0">

    <persistence-unit name="miUnidad" transaction-type="RESOURCE_LOCAL">
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

        <class>com.ejemplo.modelo.Usuario</class>
        <class>com.ejemplo.modelo.Pedido</class>

        <properties>
            <property name="jakarta.persistence.jdbc.url"
                      value="jdbc:postgresql://localhost:5432/mi_bd"/>
            <property name="jakarta.persistence.jdbc.user" value="admin"/>
            <property name="jakarta.persistence.jdbc.password" value="secreto"/>
            <property name="jakarta.persistence.jdbc.driver"
                      value="org.postgresql.Driver"/>

            <!-- Propiedades especificas de Hibernate -->
            <property name="hibernate.dialect"
                      value="org.hibernate.dialect.PostgreSQLDialect"/>
            <property name="hibernate.hbm2ddl.auto" value="validate"/>
            <property name="hibernate.show_sql" value="true"/>
        </properties>
    </persistence-unit>
</persistence>
```

### Unidad de persistencia

El atributo `name` identifica la unidad y se usa al crear el `EntityManagerFactory`: `Persistence.createEntityManagerFactory("miUnidad")`. El atributo `transaction-type` admite dos valores:

- `RESOURCE_LOCAL`: gestion de transacciones mediante JDBC directo. El desarrollador controla `begin()`, `commit()` y `rollback()`. Tipico en aplicaciones Java SE o Spring Boot con `spring-boot-starter-data-jpa` (aunque Spring abstrae esta gestion).

- `JTA`: transacciones gestionadas por el contenedor Jakarta EE. El servidor de aplicaciones coordina las transacciones, permitiendo que una unica transaccion abarque multiples recursos (base de datos, colas JMS, etc.).

### DataSource vs JDBC directo

En entornos Jakarta EE y Spring Boot es preferible declarar un `DataSource` externamente (gestionado por el servidor o por el pool de conexiones del contenedor) y referenciarlo en `persistence.xml` mediante `jta-data-source` (para JTA) o `non-jta-data-source` (para RESOURCE_LOCAL). Esto desacopla la configuracion de conexion del codigo de persistencia y permite usar pooling profesional (HikariCP, Tomcat CP).

```xml
<persistence-unit name="miUnidad" transaction-type="JTA">
    <jta-data-source>jdbc/miDataSource</jta-data-source>
    <!-- Las propiedades de conexion NO van aqui,
         se configuran en el servidor de aplicaciones -->
</persistence-unit>
```

Cuando no se dispone de un DataSource externo, se configuran las propiedades `jakarta.persistence.jdbc.url`, `user`, `password` y `driver` directamente en el XML. El `EntityManagerFactory` creara entonces sus propias conexiones, sin pool externo (lo cual no es recomendable en produccion).

### Propiedades JPA y del proveedor

Las propiedades con prefijo `jakarta.persistence` son estandar y cualquier proveedor las reconoce. Las propiedades especificas del proveedor (como `hibernate.dialect` o `eclipselink.ddl-generation`) no son portables. Al migrar de proveedor hay que revisar todas las propiedades propietarias.

| Propiedad | Descripcion |
|---|---|
| `jakarta.persistence.jdbc.url` | URL de conexion JDBC |
| `jakarta.persistence.jdbc.user` | Usuario de BD |
| `jakarta.persistence.jdbc.password` | Password |
| `jakarta.persistence.jdbc.driver` | Clase del driver JDBC |
| `jakarta.persistence.schema-generation.database.action` | Accion sobre esquema: none, create, drop-and-create, drop |

## Configuracion programatica

JPA permite construir el `EntityManagerFactory` sin XML, pasando la configuracion mediante un `Map` de propiedades. Esto es util en pruebas unitarias, aplicaciones Spring Boot (aunque Spring prefiere su autoconfiguracion) o escenarios donde la configuracion debe generarse dinamicamente.

```java
Map<String, Object> props = new HashMap<>();
props.put("jakarta.persistence.jdbc.url", "jdbc:h2:mem:testdb");
props.put("jakarta.persistence.jdbc.user", "sa");
props.put("jakarta.persistence.jdbc.password", "");
props.put("jakarta.persistence.jdbc.driver", "org.h2.Driver");
props.put("hibernate.hbm2ddl.auto", "create-drop");
props.put("hibernate.show_sql", "true");

EntityManagerFactory emf =
    new PersistenceProvider().createEntityManagerFactory("miUnidad", props);
```

El metodo `Persistence.createEntityManagerFactory` tambien acepta un `Map` de propiedades que sobrescribe las definidas en `persistence.xml`. Esto permite tener un XML base para produccion y sobrescribir con configuracion de pruebas.

## Spring Boot auto-configuration

Spring Boot simplifica la configuracion JPA hasta el punto de que `persistence.xml` deja de ser necesario. Mediante `spring-boot-starter-data-jpa`, Spring Boot autoconfigura el `DataSource` (basandose en propiedades de `application.properties` o `application.yml`), el `EntityManagerFactory` y la gestion de transacciones.

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mi_bd
    username: admin
    password: secreto
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
    open-in-view: false
```

Spring Boot crea un bean `EntityManagerFactory` configurado con estas propiedades. El desarrollador solo necesita inyectar `EntityManager` o usar `@PersistenceContext` en sus repositorios. La opcion `spring.jpa.open-in-view=false` es importante desactivarla en produccion para evitar el patron Open Session in View, que mantiene la sesion abierta durante toda la peticion web y puede causar lazy loading fuera de la transaccion.

```java
@Component
public class UsuarioService {
    private final EntityManager em;

    public UsuarioService(EntityManager em) {
        this.em = em;
    }

    @Transactional
    public void guardar(Usuario usuario) {
        em.persist(usuario);
    }
}
```

La autoconfiguracion de Spring Boot escanea el classpath en busca de clases anotadas con `@Entity` y las registra automaticamente en la unidad de persistencia. No es necesario listarlas explicitamente.
