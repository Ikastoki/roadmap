# @Transactional

La anotacion `@Transactional` es el mecanismo declarativo de Spring para gestionar transacciones. Aplicada a un metodo o a una clase, indica al contenedor que debe envolver la ejecucion en una transaccion gestionada por Spring, con las propiedades de propagacion, aislamiento y timeout especificadas. Internamente, Spring AOP intercepta la llamada al metodo, inicia una transaccion antes de ejecutar el cuerpo del metodo y realiza commit o rollback al finalizar, dependiendo de si el metodo completa con exito o lanza una excepcion.

Cuando `@Transactional` se aplica a nivel de clase, todos los metodos publicos de la clase heredan las propiedades de la transaccion. Si un metodo especifico necesita configuracion diferente, puede anotarse individualmente sobreescribiendo la configuracion de la clase. La anotacion a nivel de metodo tiene prioridad sobre la de la clase.

```java
@Service
@Transactional
public class PedidoService {

    // Hereda @Transactional de la clase con configuracion por defecto
    public Pedido crearPedido(Pedido pedido) {
        return pedidoRepository.save(pedido);
    }

    // Sobreescribe con configuracion especifica
    @Transactional(readOnly = true, timeout = 10)
    public Pedido buscarPedido(Long id) {
        return pedidoRepository.findById(id).orElseThrow();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void registrarLog(LogEntry log) {
        logRepository.save(log);
    }
}
```

> Si `@Transactional` se aplica a una clase `private`, no tiene efecto. Spring AOP solo puede interceptar metodos publicos, por lo que la anotacion solo funciona sobre metodos publicos o sobre la clase completa (cuyos metodos publicos seran interceptados).

## Propagation Types

La propagacion define como se comporta una transaccion cuando un metodo anotado con `@Transactional` es llamado desde otro metodo que ya tiene una transaccion en curso. El tipo de propagacion determina si se reutiliza la transaccion existente, se crea una nueva, o si la llamada debe o no tener una transaccion activa.

### REQUIRED (por defecto)

El tipo de propagacion por defecto. Si ya existe una transaccion en curso, el metodo se ejecuta dentro de esa transaccion. Si no existe ninguna, se crea una nueva. Es la opcion mas comun y segura para operaciones de escritura.

```
Metodo A (@Transactional REQUIRED)
    │
    ▼
Metodo B (@Transactional REQUIRED) llamado desde A
    │
    ▼
B se ejecuta dentro de la transaccion de A (misma transaccion)
    │
    ▼
Si B falla, se revierte toda la transaccion (A y B)
```

```java
@Service
public class PedidoService {

    @Transactional
    public Pedido crearPedido(Pedido pedido) {
        Pedido guardado = pedidoRepository.save(pedido);
        // Esta llamada reutiliza la misma transaccion
        inventarioService.descontarStock(guardado.getProductoId(), guardado.getCantidad());
        return guardado;
    }
}

@Service
public class InventarioService {

    @Transactional  // REQUIRED por defecto: reutiliza la transaccion de crearPedido
    public void descontarStock(Long productoId, int cantidad) {
        // se ejecuta en la misma transaccion
    }
}
```

### REQUIRES_NEW

Siempre crea una nueva transaccion, independientemente de si existe una en curso. La transaccion existente se suspende hasta que la nueva termine. Si la nueva transaccion hace commit, los cambios son visibles incluso si la transaccion original luego falla. Si la nueva hace rollback, no afecta a la transaccion suspendida.

```
Metodo A (@Transactional REQUIRED) - transaccion T1 activa
    │
    ▼
Metodo B (@Transactional REQUIRES_NEW) - crea transaccion T2
    │
    ├── T2 commitea → cambios de B persisten
    │
    ▼
T1 continua despues de que T2 termina
    │
    ├── Si T1 commitea → ambos commits son efectivos
    └── Si T1 falla → solo T1 se revierte, T2 no
```

```java
@Service
public class PedidoService {

    @Transactional
    public Pedido crearPedido(Pedido pedido) {
        Pedido guardado = pedidoRepository.save(pedido);
        // REQUIRES_NEW: crea transaccion aparte para el log
        logService.registrarAccion("Pedido creado: " + guardado.getId());
        if (error) {
            throw new RuntimeException("Error ficticio");
        }
        return guardado;
    }
}

@Service
public class LogService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void registrarAccion(String accion) {
        // Esta transaccion es independiente
        // Se commitea aunque PedidoService haga rollback
        logRepository.save(new LogEntry(accion));
    }
}
```

> REQUIRES_NEW es ideal para registrar logs de auditoria o metricas que deben persistirse incluso si la operacion principal falla. Sin embargo, implica una segunda transaccion con su propio overhead de commit.

### NESTED

Crea un savepoint dentro de la transaccion existente. Si la transaccion interna falla, solo se revierte hasta el savepoint, no la transaccion completa. La transaccion padre puede decidir si commitea o revierte todo. Solo es soportado por proveedores que manejan savepoints, como JDBC con autocommit deshabilitado o JPA.

```
Metodo A (@Transactional REQUIRED) - transaccion T1
    │
    ▼
Metodo B (@Transactional NESTED) - savepoint S1 dentro de T1
    │
    ├── Si B falla → se revierte hasta S1, T1 sigue activa
    │
    ▼
A continua ejecutando en T1
    │
    ├── Si A commitea → se libera S1 y T1 se commitea
    └── Si A falla → se revierte todo T1 incluyendo S1
```

```java
@Service
public class ProcesamientoService {

    @Transactional
    public void procesarLote(List<Item> items) {
        for (Item item : items) {
            try {
                procesarItem(item);  // NESTED: si falla un item, el lote sigue
            } catch (Exception e) {
                logger.warn("Error procesando item: " + item.getId());
            }
        }
    }

    @Transactional(propagation = Propagation.NESTED)
    public void procesarItem(Item item) {
        repository.save(item);
    }
}
```

### SUPPORTS

Si hay una transaccion en curso, se ejecuta dentro de ella. Si no hay ninguna, se ejecuta sin transaccion (autocommit). Es util para metodos de lectura que pueden funcionar con o sin transaccion.

### NOT_SUPPORTED

Se ejecuta siempre sin transaccion. Si hay una en curso, se suspende. Es util para operaciones que no deben formar parte de una transaccion, como llamadas a servicios externos lentos.

### MANDATORY

Debe ejecutarse dentro de una transaccion existente. Si no hay ninguna activa, lanza `TransactionRequiredException`. Se usa cuando el metodo siempre necesita transaccion y la responsabilidad de crearla es del caller.

### NEVER

Nunca debe ejecutarse dentro de una transaccion. Si hay una activa, lanza `TransactionException`. Se usa para operaciones que no son transaccionales por naturaleza y que podrian causar problemas si se ejecutaran dentro de una.

```
Resumen de propagaciones:

REQUIRED       → Reusa existente o crea nueva (default)
REQUIRES_NEW   → Siempre crea nueva (suspende la actual)
NESTED         → Savepoint dentro de la transaccion actual
SUPPORTS       → Reusa existente o ejecuta sin transaccion
NOT_SUPPORTED  → Ejecuta sin transaccion (suspende la actual)
MANDATORY      → Requiere transaccion existente (error si no hay)
NEVER          → No admite transaccion (error si la hay)
```

## Isolation Levels

El nivel de aislamiento determina el grado en que las transacciones concurrentes estan aisladas entre si. Un nivel mas alto ofrece mayor consistencia pero reduce la concurrencia (mas bloqueos, menor rendimiento). El nivel por defecto depende del proveedor de base de datos: MySQL usa `REPEATABLE_READ`, PostgreSQL y Oracle usan `READ_COMMITTED`.

### READ_UNCOMMITTED

El nivel mas bajo. Permite leer datos modificados por otra transaccion que aun no ha commiteado (lecturas sucias). Ofrece la mejor concurrencia pero la peor integridad. Es raramente util en aplicaciones de negocio.

### READ_COMMITTED

Solo permite leer datos que han sido commiteados. Evita las lecturas sucias pero no las lecturas no repetibles: si otra transaccion modifica un registro entre dos lecturas de la misma transaccion, los valores pueden ser diferentes. Es el nivel por defecto de PostgreSQL y Oracle.

### REPEATABLE_READ

Garantiza que si una transaccion lee un registro dos veces, obtendra los mismos valores ambas veces. Otras transacciones no pueden modificar los registros leidos hasta que la transaccion actual termine. No previene phantom reads (nuevos registros insertados por otras transacciones que coinciden con una consulta).

### SERIALIZABLE

El nivel mas alto. Ejecuta las transacciones como si fueran secuenciales, no concurrentes. Previne dirty reads, non-repeatable reads y phantom reads, pero con un costo significativo en rendimiento. Solo se usa cuando la consistencia es critica y la concurrencia es baja.

> En la mayoria de aplicaciones web, `READ_COMMITTED` ofrece el mejor equilibrio entre consistencia y rendimiento. `REPEATABLE_READ` se reserva para operaciones financieras o de inventario donde la lectura consistente es fundamental.

## timeout y readOnly

El atributo `timeout` establece el tiempo maximo en segundos que una transaccion puede permanecer abierta. Si se excede, la transaccion se revierte automaticamente y se lanza una `TransactionTimedOutException`. Por defecto es -1 (sin limite), lo que depende del proveedor de base de datos.

```java
@Transactional(timeout = 30)  // maximo 30 segundos
public void procesamientoLargo() {
    // si tarda mas de 30s, se revierte
}
```

El atributo `readOnly` indica que la transaccion solo realizara operaciones de lectura. No es solo documentacion: el proveedor puede optimizar internamente (no adquirir locks de escritura, usar Read Only Transaction en Hibernate, etc.). Debe usarse siempre que el metodo no modifique datos.

```java
@Transactional(readOnly = true)
public List<Usuario> buscarUsuarios(String filtro) {
    return usuarioRepository.findByNombreContaining(filtro);
}
```

## rollbackFor y noRollbackFor

Por defecto, Spring realiza rollback solo ante `RuntimeException` y `Error`. Las checked exceptions (`Exception` y sus subclases que no son `RuntimeException`) no provocan rollback, lo cual puede ser contraintuitivo. El atributo `rollbackFor` permite especificar que tipos de excepcion deben provocar rollback.

```java
// Rollback para una excepcion checked especifica
@Transactional(rollbackFor = MiExcepcionChecked.class)
public void metodoQuePuedeLanzarChecked() throws MiExcepcionChecked {
    repository.save(entity);
    if (error) {
        throw new MiExcepcionChecked("Error controlado");
    }
}

// No hacer rollback para una excepcion unchecked especifica
@Transactional(noRollbackFor = ExcepcionNoCritica.class)
public void metodoConExcepcionTolerable() {
    repository.save(entity);
    if (errorMenor) {
        throw new ExcepcionNoCritica("No es critico");
    }
    // la transaccion se commitea aunque se lance ExcepcionNoCritica
}
```

> Una practica recomendada es especificar explicitamente `rollbackFor = Exception.class` cuando se quiere que cualquier excepcion (checked o unchecked) provoque rollback: `@Transactional(rollbackFor = Exception.class)`.

## @Transactional en Spring Boot

Spring Boot simplifica la configuracion de transacciones de varias formas. La autoconfiguracion detecta automaticamente el `PlatformTransactionManager` adecuado, configura el proxy de AOP y habilita la anotacion `@Transactional`. Para activar el soporte se necesita `spring-boot-starter-data-jpa` o `spring-boot-starter-jdbc` en las dependencias.

```properties
# Configuracion de transacciones en application.properties
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.open-in-view=false
```

La propiedad `spring.jpa.open-in-view=false` es recomendada en produccion. Cuando esta habilitada (por defecto en Spring Boot 2.x), mantiene la transaccion abierta durante toda la vida del request HTTP, lo que puede causar lazy loading issues y bloqueos prolongados en la base de datos. Deshabilitarla fuerza a que las operaciones de persistencia se realicen dentro de la transaccion de servicio, con un ciclo de vida mas corto y predecible.

```java
@SpringBootApplication
@EnableTransactionManagement  // habilitado por defecto en Boot, opcional explicitarlo
public class MiAplicacion {
    public static void main(String[] args) {
        SpringApplication.run(MiAplicacion.class, args);
    }
}
```

> `@EnableTransactionManagement` esta habilitado por defecto en Spring Boot. Solo es necesario incluirlo explicitamente si se necesita configurar algun atributo como `mode`, `proxyTargetClass` o `order`.

