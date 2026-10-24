# Configuracion de H2

La integracion de H2 en un proyecto Java requiere anadir la dependencia correspondiente y configurar la URL de conexion segun el modo de operacion deseado. H2 ofrece multiples opciones de conexion que determinan donde y como se almacenan los datos, asi como el alcance del ciclo de vida de la base de datos.

## Dependencia Maven

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.2.224</version>
    <scope>test</scope>
</dependency>
```

En proyectos que utilizan H2 tambien para desarrollo (no solo testing), el scope puede omitirse o cambiarse a `runtime`. La version 2.x introdujo cambios importantes respecto a la rama 1.4.x, como la eliminacion del motor MVCC antiguo y la adopcion del motor MVStore por defecto.

## Dependencia Gradle

```groovy
testImplementation 'com.h2database:h2:2.2.224'
```

## URL de conexion

La URL de conexion es el parametro fundamental que define el comportamiento de H2. Sigue el formato `jdbc:h2:[modo]:[ruta]` y admite multiples parametros adicionales separados por punto y coma.

### Memoria (modo embebido volatil)

```java
// Solo un cliente en la misma JVM
Connection conn = DriverManager.getConnection("jdbc:h2:mem:testdb");

// Varios clientes con el mismo nombre (misma JVM)
Connection conn = DriverManager.getConnection("jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1");
```

En modo memoria, la base de datos se destruye al cerrar la ultima conexion. El parametro `DB_CLOSE_DELAY=-1` evita que se cierre automaticamente, permitiendo que multiples conexiones compartan la misma base de datos en memoria dentro de la misma JVM.

### Disco (modo embebido persistente)

```java
// Ruta relativa al directorio de trabajo
Connection conn = DriverManager.getConnection("jdbc:h2:file:./data/mibd");

// Ruta absoluta en el sistema de archivos
Connection conn = DriverManager.getConnection("jdbc:h2:file:/home/usuario/datos/mibd");

// Directorio del usuario (~ se expande al home)
Connection conn = DriverManager.getConnection("jdbc:h2:~/datos/mibd");
```

En modo archivo, H2 crea los siguientes ficheros:
- `mibd.mv.db`: archivo principal de datos (formato MVStore).
- `mibd.trace.db`: archivo de traza con las operaciones ejecutadas.
- `mibd.lock.db`: archivo de bloqueo para evitar accesos concurrentes.

### Servidor TCP

```java
Connection conn = DriverManager.getConnection("jdbc:h2:tcp://localhost:9092/~/mibd");
```

Requiere que el servidor H2 este previamente iniciado. El simbolo `~` se expande al directorio home del usuario que ejecuta el servidor.

### Modo de compatibilidad

```java
Connection conn = DriverManager.getConnection(
    "jdbc:h2:mem:testdb;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE"
);
```

Los modos de compatibilidad ajustan funciones, tipos de datos y comportamiento SQL. Algunos parametros utiles son:
- `MODE=PostgreSQL` o `MODE=MySQL`
- `DATABASE_TO_LOWER=TRUE`: convierte automaticamente los nombres de objetos a minusculas, como hace PostgreSQL.

## Configuracion en Spring Boot

Spring Boot simplifica la configuracion de H2 mediante propiedades en `application.properties` o `application.yml`.

```properties
# application.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# Inicializacion del esquema
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driverClassName: org.h2.Driver
    username: sa
    password: ""
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
  h2:
    console:
      enabled: true
      path: /h2-console
```

La propiedad `spring.jpa.hibernate.ddl-auto` admite los siguientes valores:
- `none`: no realiza ninguna accion sobre el esquema.
- `create`: crea el esquema al arrancar, pero no lo elimina al detener.
- `create-drop`: crea el esquema al arrancar y lo elimina al detener (ideal para testing en memoria).
- `update`: actualiza el esquema existente sin destruir datos.
- `validate`: solo verifica que el esquema coincide con las entidades.

> En produccion jamas debe usarse `create-drop` o `update` con una base de datos real. Estos valores pueden provocar perdida de datos o cambios inesperados en el esquema. Su uso queda restringido a entornos de desarrollo y testing.

Para perfiles especificos, se pueden definir configuraciones separadas. Por ejemplo, un perfil `test` con H2 en memoria y un perfil `dev` con PostgreSQL:

```properties
# application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
spring.jpa.hibernate.ddl-auto=create-drop
```

> El parametro `DB_CLOSE_DELAY=-1` es especialmente util en testing cuando se usa `DataSource` con pool de conexiones. Sin este parametro, H2 podria cerrar la base de datos al liberar la primera conexion, causando errores en las siguientes.
