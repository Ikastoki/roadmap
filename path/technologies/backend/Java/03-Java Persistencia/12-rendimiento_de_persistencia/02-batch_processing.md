# Batch Processing en persistencia

El procesamiento por lotes (batch processing) es una tecnica fundamental para mejorar el rendimiento de operaciones masivas de insercion, actualizacion o eliminacion de datos. Sin batch processing, Hibernate ejecuta cada operacion en una sentencia SQL independiente, lo que genera una sobrecarga enorme en terminos de rondas de ida y vuelta a la base de datos, planificacion de consultas y escritura en logs de transacciones. Con batch processing, multiples operaciones se agrupan en una unida sentencia SQL o en un unico lote enviado a la base de datos.

## JDBC batch operations

A nivel de JDBC, el batch processing se configura mediante la propiedad `rewriteBatchedInserts` en la URL de conexion (PostgreSQL) o `rewriteBatchedStatements` (MySQL):

```properties
# PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb?reWriteBatchedInserts=true

# MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/mydb?rewriteBatchedStatements=true
```

Sin estas propiedades, JDBC ejecuta cada INSERT como una sentencia individual, anulando cualquier configuracion de batch a nivel de Hibernate.

## Configuracion de Hibernate batch

Hibernate requiere tres propiedades fundamentales para habilitar el batch processing:

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

`batch_size` define el numero maximo de sentencias que Hibernate agrupa en un lote antes de enviarlas a la base de datos. Un valor entre 20 y 50 suele ser optimo. Valores muy altos pueden saturar el buffer de la base de datos sin mejorar el rendimiento.

`order_inserts=true` y `order_updates=true` instruyen a Hibernate para reordenar las sentencias SQL de forma que las operaciones sobre la misma tabla se agrupen juntas. Sin esta reordenacion, Hibernate mezcla las sentencias de diferentes tablas y JDBC no puede agruparlas en un unico lote.

```java
@Test
void insercionMasivaSinBatch() {
    for (int i = 0; i < 1000; i++) {
        Usuario usuario = new Usuario("user" + i + "@email.com", "Nombre", "Apellido");
        usuarioRepository.save(usuario);
    }
    // Sin batch: 1000 INSERTs individuales
    // Con batch_size=50 y order_inserts=true: 20 lotes de 50 INSERTs
}
```

## Estrategia de ids para batch

La generacion de identificadores tiene un impacto directo en la eficiencia del batch processing. El uso de `GenerationType.IDENTITY` impide que Hibernate agrupe INSERTs, porque necesita conocer el ID generado inmediatamente despues de cada insercion para gestionar el contexto de persistencia.

```java
@Entity
public class Usuario {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // NO USAR con batch
    private Long id;
}
```

Las estrategias compatibles con batch son `GenerationType.SEQUENCE` y `GenerationType.TABLE`:

```java
@Entity
public class Usuario {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "usuario_seq")
    @SequenceGenerator(name = "usuario_seq", sequenceName = "usuario_seq", allocationSize = 50)
    private Long id;
}
```

`allocationSize` debe coincidir o ser multiplo de `batch_size` para evitar llamadas innecesarias a la secuencia. Con `allocationSize=50` y `batch_size=50`, Hibernate solicita 50 IDs de una sola vez y puede ejecutar el lote completo sin pausas.

## Flushing y limpieza del PersistenceContext

Durante un procesamiento masivo, el `PersistenceContext` de Hibernate (la cache de primer nivel) acumula todas las entidades gestionadas. Si no se limpia periodicamente, la memoria se agota y el rendimiento se degrada por la sobrecarga de gestion de entidades en memoria.

```java
@Transactional
public void insertarMasivo(int cantidad) {
    Session session = entityManager.unwrap(Session.class);

    for (int i = 0; i < cantidad; i++) {
        Usuario usuario = new Usuario("user" + i + "@email.com", "Nombre", "Apellido");
        session.persist(usuario);

        if (i % 50 == 0) {
            session.flush();    // Envia el lote a la base de datos
            session.clear();    // Libera memoria del PersistenceContext
        }
    }
}
```

`flush()` sincroniza el estado del `PersistenceContext` con la base de datos, forzando la ejecucion del lote pendiente. `clear()` elimina todas las entidades del contexto de persistencia, liberando memoria y evitando que Hibernate tenga que hacer comprobaciones de suciedad sobre entidades que ya no se van a modificar.

## Batch updates y deletes

Las operaciones de actualizacion y eliminacion masivas tambien se benefician del batch processing. Ademas de `batch_size` y `order_updates=true`, se puede utilizar una consulta `@Modifying` para evitar cargar las entidades en memoria:

```java
@Modifying
@Query("UPDATE Usuario u SET u.activo = false WHERE u.ultimoAcceso < :fechaLimite")
int desactivarUsuariosInactivos(@Param("fechaLimite") LocalDateTime fechaLimite);
```

Este enfoque ejecuta una unica sentencia SQL en lugar de cargar N entidades y ejecutar N UPDATEs individuales.

## Rendimiento 10x-100x

El impacto del batch processing es dramatico. Sin batch, insertar 10000 registros puede requerir 10000 viajes de ida y vuelta a la base de datos. Con `batch_size=50`, se reducen a 200 viajes. La medicion tipica muestra mejoras de 10x a 100x en operaciones masivas:

| Configuracion | Tiempo (10000 inserts) |
|---------------|----------------------|
| Sin batch | 45 segundos |
| batch_size=50 | 3 segundos |
| batch_size=50 + order_inserts | 1.5 segundos |
| + reWriteBatchedInserts | 0.8 segundos |

> El batch processing requiere JDBC 2.0 o superior (disponible desde Java 1.2). Todas las bases de datos modernas lo soportan, pero la propiedad de URL para reescritura de batches varia segun el motor. PostgreSQL usa `reWriteBatchedInserts=true`, mientras que MySQL requiere `rewriteBatchedStatements=true`.

## StatelessSession para carga masiva

Hibernate proporciona `StatelessSession` como alternativa para operaciones masivas donde no se necesita el contexto de persistencia ni las comprobaciones de suciedad:

```java
@Transactional
public void insertarConStatelessSession(int cantidad) {
    StatelessSession session = sessionFactory.openStatelessSession();
    session.setJdbcBatchSize(50);
    Transaction tx = session.beginTransaction();

    for (int i = 0; i < cantidad; i++) {
        Usuario usuario = new Usuario("user" + i + "@email.com", "Nombre", "Apellido");
        session.insert(usuario);
    }

    tx.commit();
    session.close();
}
```

`StatelessSession` no mantiene cache de primer nivel, no gestiona el ciclo de vida de las entidades ni ejecuta comprobaciones de suciedad. Esto la hace significativamente mas rapida para inserciones masivas, pero tampoco proporciona las garantias de coherencia del `PersistenceContext`.
