# Testing con H2

El testing de la capa de persistencia es una parte fundamental en el desarrollo de aplicaciones Spring. Spring Boot proporciona un soporte excelente para pruebas de integracion con bases de datos mediante H2, una base de datos relacional escrita en Java que puede ejecutarse en modo embebido. H2 es ideal para testing porque arranca en milisegundos, no requiere instalacion externa y ofrece compatibilidad con el dialecto SQL de los principales motores de produccion.

## @DataJpaTest

`@DataJpaTest` es la anotacion central para pruebas de persistencia en Spring Boot. Configura unicamente los componentes relacionados con JPA: `EntityManager`, `DataSource`, repositorios y el esquema de la base de datos. No carga el resto de la aplicacion (controladores, servicios, etc.), lo que hace que los tests sean rapidos y enfocados.

```java
@DataJpaTest
class UsuarioRepositoryTest {

    @Autowired
    private UsuarioRepository usuarioRepository;

    @Test
    void deberiaGuardarYEncontrarUsuario() {
        Usuario usuario = new Usuario("juan@email.com", "Juan", "Perez");
        usuarioRepository.save(usuario);

        Optional<Usuario> encontrado = usuarioRepository.findById(usuario.getId());

        assertThat(encontrado).isPresent();
        assertThat(encontrado.get().getEmail()).isEqualTo("juan@email.com");
    }
}
```

`@DataJpaTest` configura automaticamente una base de datos H2 en memoria para cada test. Al finalizar cada metodo, la transaccion se revierte y los datos se descartan, garantizando el aislamiento entre pruebas.

> `@DataJpaTest` solo escanea las entidades JPA y los repositorios. Si el test necesita componentes de otras capas (servicios, controladores), debe anadirse `@AutoConfigureTestDatabase` o usar `@SpringBootTest`.

## @AutoConfigureTestDatabase

`@AutoConfigureTestDatabase` reemplaza el `DataSource` configurado para produccion por una base de datos embebida (H2 por defecto). Esto permite que los tests se ejecuten contra H2 incluso si la aplicacion en produccion usa PostgreSQL o MySQL.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.ANY)
class UsuarioRepositoryTest {
    // ...
}
```

`Replace.ANY` (valor por defecto) sustituye cualquier `DataSource` configurado. `Replace.NONE` mantiene el `DataSource` original, util cuando se quiere probar contra la base de datos real.

Si se desea mantener la compatibilidad con la base de datos de produccion, se puede configurar el dialecto H2 para que emule el motor objetivo:

```properties
spring.datasource.url=jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.H2Dialect
```

El parametro `MODE=PostgreSQL` hace que H2 se comporte de forma mas parecida a PostgreSQL en terminos de funciones SQL, manejo de tipos y sintaxis.

## Datos iniciales con data.sql y schema.sql

Spring Boot ejecuta automaticamente los archivos `schema.sql` y `data.sql` en el classpath al iniciar el contexto de pruebas. `schema.sql` crea las tablas y `data.sql` inserta datos iniciales.

`schema.sql`:

```sql
CREATE TABLE usuarios (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    nombre VARCHAR(100) NOT NULL,
    apellido VARCHAR(100) NOT NULL
);
```

`data.sql`:

```sql
INSERT INTO usuarios (email, nombre, apellido) VALUES ('admin@email.com', 'Admin', 'Principal');
INSERT INTO usuarios (email, nombre, apellido) VALUES ('user@email.com', 'Usuario', 'Normal');
```

Si se utiliza JPA con `ddl-auto` (por ejemplo, `spring.jpa.hibernate.ddl-auto=create-drop`), Hibernate genera automaticamente las tablas a partir de las entidades. En ese caso, incluir `schema.sql` provocaria errores porque las tablas ya existen. Para evitar conflictos se debe deshabilitar la generacion automatica o configurar `spring.jpa.defer-datasource-initialization=true`.

```properties
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.defer-datasource-initialization=true
```

## import.sql

Como alternativa a `data.sql`, Hibernate reconoce el archivo `import.sql` en el classpath. Si `ddl-auto` esta configurado para crear el esquema (`create`, `create-drop`), Hibernate ejecuta `import.sql` automaticamente despues de crear las tablas:

```sql
INSERT INTO usuarios (email, nombre, apellido) VALUES ('test@email.com', 'Test', 'Usuario');
```

La ventaja de `import.sql` sobre `data.sql` es que no requiere configuracion adicional de deferimiento. La desventaja es que solo se ejecuta cuando Hibernate crea el esquema, no cuando se usa `schema.sql`.

## Configuracion de application-test.properties

Para configuraciones especificas del perfil de test, se puede crear `application-test.properties` en `src/test/resources`:

```properties
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

Y activar el perfil en el test con `@ActiveProfiles("test")`.

> `spring.jpa.show-sql=true` muestra en la consola las sentencias SQL generadas por Hibernate. Es util durante el desarrollo para verificar que las consultas generadas son las esperadas, pero debe desactivarse en produccion por razones de rendimiento y seguridad.
