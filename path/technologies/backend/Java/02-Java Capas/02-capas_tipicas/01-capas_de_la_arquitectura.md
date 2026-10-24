# Capas de la arquitectura

Una aplicacion empresarial raramente es un monolito sin estructura interna. Al crecer, el acoplamiento entre componentes convierte cualquier cambio en una operacion de alto riesgo. La separacion en capas responde a esa tension: divide el software en ambitos horizontales donde cada estrato se comunica unicamente con el inmediato inferior, estableciendo una jerarquia de dependencias que permite modificar, probar y desplegar cada nivel con un impacto predecible sobre los demas.

## Capa de presentacion

La capa de presentacion es el punto de entrada del sistema. Su responsabilidad se limita a recibir peticiones externas (HTTP, mensajes de cola, eventos de interfaz de usuario), validar la sintaxis de los datos entrantes y delegar el procesamiento en las capas inferiores. Una vez que la capa de negocio devuelve un resultado, la presentacion lo formatea en la representacion adecuada para el cliente: JSON, XML, HTML o cualquier otro formato de serializacion.

En el ecosistema Java esta capa se materializa en controladores REST (anotados con `@RestController` en Spring MVC o `@Path` en JAX-RS), servlets tradicionales o componentes de frameworks web como JSF. Un controlador tipico declara un endpoint, extrae los parametros de la request, invoca un servicio y devuelve una respuesta envuelta en el codigo HTTP apropiado. No debe contener condicionales que reflejen reglas de negocio ni acceso directo a la base de datos.

```java
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController {

    private final UsuarioService service;

    public UsuarioController(UsuarioService service) {
        this.service = service;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UsuarioDTO> obtener(@PathVariable Long id) {
        UsuarioDTO dto = service.obtenerPorId(id);
        return ResponseEntity.ok(dto);
    }
}
```

La validacion que ocurre aqui es exclusivamente estructural: que un campo obligatorio no sea nulo, que un email tenga formato correcto, que un identificador sea numerico. Las validaciones semanticas, como comprobar si un usuario tiene saldo suficiente para realizar una operacion, pertenecen a la capa de negocio.

> La capa de presentacion es la unica que deberia depender directamente de teconologias de transporte y serializacion (HTTP, JSON, XML). Cambiar de REST a GraphQL o de JSON a protobuf deberia implicar modificar solo esta capa.

## Capa de negocio

La capa de negocio contiene las reglas que definen el proposito del sistema. Aqui se implementan los casos de uso: calcular un precio, validar una transferencia, autorizar un acceso, orquestar un flujo de aprobacion. Es el estrato que diferencia una simple CRUD de una aplicacion con valor real.

En Java esta capa se concreta en servicios (clases `@Service` en Spring), casos de uso o interactores siguiendo el patron _Clean Architecture_. Un servicio recibe datos ya validados estructuralmente, aplica las reglas de negocio y coordina las operaciones con los repositorios necesarios. La gestion transaccional se declara tipicamente a este nivel mediante `@Transactional`, asegurando que multiples operaciones de escritura se ejecuten como una unidad atomica.

```java
@Service
public class TransferenciaService {

    private final CuentaRepository cuentas;
    private final NotificacionService notificaciones;

    @Transactional
    public ResultadoTransferencia ejecutar(OrdenTransferencia orden) {
        Cuenta origen = cuentas.buscarPorId(orden.getCuentaOrigenId());
        Cuenta destino = cuentas.buscarPorId(orden.getCuentaDestinoId());
        origen.debitar(orden.getImporte());
        destino.acreditar(orden.getImporte());
        cuentas.guardar(origen);
        cuentas.guardar(destino);
        notificaciones.enviarComprobante(origen.getTitular(), orden);
        return new ResultadoTransferencia(true, "Transferencia exitosa");
    }
}
```

La capa de negocio no debe tener dependencias con frameworks de presentacion ni con detalles de persistencia. Trabaja con modelos de dominio (o DTOs de negocio) y define interfaces para los repositorios que necesita; la implementacion concreta de esas interfaces se inyecta desde el exterior.

## Capa de acceso a datos

La capa de acceso a datos abstrae el mecanismo de persistencia subyacente. Su funcion es traducir operaciones del lenguaje del dominio (guardar un usuario, buscar una factura por rango de fechas) a instrucciones que la base de datos entiende. Implementa el patron **Repository**, que presenta a la capa de negocio una interfaz similar a una coleccion en memoria, ocultando los detalles de SQL, conexiones, transacciones de bajo nivel y mapeo objeto-relacional.

En Java esta capa se construye con repositorios JPA (extendiendo `JpaRepository` de Spring Data), DAOs clasicos con `EntityManager` o clases que usan `JdbcTemplate` para acceso directo a SQL. Independientemente de la tecnologia, el contrato con la capa superior son interfaces: la capa de negocio programa contra esas interfaces y desconoce si detras hay una base relacional, un caché Redis o un servicio externo.

```java
@Repository
public class UsuarioRepositoryImpl implements UsuarioRepository {

    private final JdbcTemplate jdbc;
    private final UsuarioRowMapper mapper;

    public UsuarioRepositoryImpl(JdbcTemplate jdbc, UsuarioRowMapper mapper) {
        this.jdbc = jdbc;
        this.mapper = mapper;
    }

    @Override
    public Usuario buscarPorId(Long id) {
        String sql = "SELECT id, nombre, email, saldo FROM usuarios WHERE id = ?";
        return jdbc.queryForObject(sql, mapper, id);
    }
}
```

> La frontera entre la capa de negocio y la de acceso a datos es donde se define el puerto (interface) y el adaptador (implementacion). Esta separacion permite cambiar de tecnica de persistencia sin modificar una sola linea de los servicios.

Los mapeadores (o _mappers_) forman parte tambien de esta capa: convierten filas de base de datos en objetos de dominio y viceversa, aislando a las capas superiores de la estructura fisica del esquema.

## Capa de base de datos

La capa de base de datos es el nivel fisico del sistema. Comprende el motor de base de datos relacional (PostgreSQL, MySQL, Oracle), sus tablas, indices, restricciones de integridad referencial, vistas materializadas, procedimientos almacenados y el esquema general de datos. A diferencia de las capas anteriores, que viven en el codigo de la aplicacion, esta capa reside en el servidor de base de datos y se gestiona mediante migraciones.

Las migraciones (ejecutadas con herramientas como Flyway o Liquibase) son el mecanismo que versiona el esquema y lo sincroniza entre entornos. Cada migracion describe un cambio incremental: crear una tabla, anadir una columna, insertar datos de referencia. El estado final de la base de datos se reconstruye aplicando las migraciones en orden secuencial desde el estado inicial.

```sql
-- Migration V001__crear_tabla_usuarios.sql
CREATE TABLE usuarios (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(150) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    saldo DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_usuarios_email ON usuarios(email);
```

Las _stored procedures_ y _triggers_ residen en esta capa cuando encapsulan logica que debe ejecutarse en el motor mismo, aunque su uso debe evaluarse con cuidado: cualquier regla de negocio implementada en un procedimiento almacenado queda fuera del control de versiones de la aplicacion y dificulta las pruebas automatizadas. La tendencia moderna es mantener la logica de negocio en la aplicacion y limitar la base de datos a almacenamiento, integridad y optimizacion de consultas.

> Una decision arquitectonica clave es donde trazar la linea entre la capa de acceso a datos y la base de datos. Las constraints de base de datos (claves foraneas, unique, check) son redundancias deliberadas: la aplicacion deberia validar antes de insertar, pero la base de datos actua como ultimo guardian de la integridad.
