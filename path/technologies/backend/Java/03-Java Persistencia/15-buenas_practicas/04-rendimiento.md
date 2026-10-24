# Rendimiento - buenas practicas

El rendimiento de la capa de persistencia determina la capacidad de respuesta de toda la aplicacion. Una consulta ineficiente, una configuracion inadecuada de fetching o un pool de conexiones mal dimensionado pueden convertir una aplicacion bien diseñada en un sistema lento e inestable. Optimizar la persistencia implica entender como Hibernate interactua con la base de datos en cada operacion y aplicar patrones que minimicen el trafico de datos y el tiempo de ejecucion.

## LAZY por defecto

Todas las asociaciones entre entidades deben configurarse como `FetchType.LAZY`. La carga perezosa difiere la recuperacion de los datos asociados hasta que se accede a ellos por primera vez, evitando que una consulta simple arrastre todo el grafo de objetos.

```java
@Entity
@Table(name = "pedidos")
public class Pedido {

    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id")
    private Cliente cliente;

    @OneToMany(mappedBy = "pedido", fetch = FetchType.LAZY)
    private List<LineaPedido> lineas;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "pedido_etiquetas")
    private Set<Etiqueta> etiquetas;
}
```

> Hibernate utiliza proxies para implementar LAZY. Si se accede a una asociacion lazy fuera de una transaccion, se lanza `LazyInitializationException`. Por eso es importante definir bien los limites transaccionales y usar DTOs cuando se necesite exponer datos asociados en la capa de presentacion.

## JOIN FETCH cuando se necesite

Cuando se sabe que una asociacion lazy va a ser necesaria en un contexto determinado, se debe cargar explicitamente mediante `JOIN FETCH` en la consulta JPQL. Esto genera un `JOIN` SQL que recupera todos los datos en una unica consulta, evitando el problema N+1.

```java
@Query("""
    SELECT p FROM Pedido p
    JOIN FETCH p.cliente
    JOIN FETCH p.lineas l
    JOIN FETCH l.producto
    WHERE p.id = :id
    """)
Optional<Pedido> findByIdConDetalles(@Param("id") Long id);
```

Sin `JOIN FETCH`, acceder a las lineas de un pedido generaria una consulta adicional por cada pedido recuperado (problema N+1). Con `JOIN FETCH`, una sola consulta con `JOIN` recupera toda la informacion necesaria.

```java
// Sin JOIN FETCH: 1 consulta para pedidos + N consultas para lineas = N+1
List<Pedido> pedidos = pedidoRepository.findAll();
for (Pedido p : pedidos) {
    System.out.println(p.getLineas().size()); // N consultas adicionales
}

// Con JOIN FETCH: 1 consulta con JOIN
List<Pedido> pedidos = pedidoRepository.findAllConLineas();
```

Otra alternativa es usar `@EntityGraph` para definir grafos de entidades que se cargaran juntos:

```java
@Entity
@Table(name = "pedidos")
@NamedEntityGraph(
    name = "pedido.con-detalles",
    attributeNodes = {
        @NamedAttributeNode("cliente"),
        @NamedAttributeNode(value = "lineas", subgraph = "lineas.producto")
    },
    subgraphs = {
        @NamedSubgraph(name = "lineas.producto",
            attributeNodes = @NamedAttributeNode("producto"))
    }
)
public class Pedido { ... }

// En el repositorio
@EntityGraph("pedido.con-detalles")
Optional<Pedido> findById(Long id);
```

## Batch operations

Procesar colecciones grandes con operaciones individuales es extremadamente ineficiente. Hibernate proporciona mecanismos de batch para agrupar multiples operaciones en pocas sentencias SQL.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 30
          batch_versioned_data: true
        order_inserts: true
        order_updates: true
```

- `batch_size`: numero de operaciones que se agrupan en un unico `INSERT` o `UPDATE` JDBC.
- `order_inserts` y `order_updates`: reordenan las sentencias para maximizar el agrupamiento.

```java
@Transactional
public void importarUsuarios(List<UsuarioDTO> usuarios) {
    for (int i = 0; i < usuarios.size(); i++) {
        UsuarioDTO dto = usuarios.get(i);
        entityManager.persist(new Usuario(dto.getEmail(), dto.getNombre()));

        // Flush y clear periodico para liberar memoria del contexto de persistencia
        if (i % 30 == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

El `flush()` sincroniza el lote actual con la base de datos y `clear()` libera las entidades del contexto de persistencia, evitando que la memoria del servidor se sature con objetos gestionados. Sin `clear()`, el contexto de persistencia crece indefinidamente en operaciones batch.

## Paginacion

Nunca se deben cargar colecciones completas en memoria si el resultado puede ser grande. Spring Data JPA proporciona `Pageable` y `Slice` para implementar paginacion de forma declarativa:

```java
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    Page<Pedido> findByClienteId(Long clienteId, Pageable pageable);

    @Query("SELECT p FROM Pedido p JOIN FETCH p.cliente WHERE p.estado = :estado")
    Slice<Pedido> findByEstado(@Param("estado") String estado, Pageable pageable);
}

// Uso en servicio
public Page<PedidoDTO> listarPedidos(int pagina, int tamanio) {
    Pageable pageable = PageRequest.of(pagina, tamanio, Sort.by("fecha").descending());
    Page<Pedido> pedidos = pedidoRepository.findByClienteId(clienteId, pageable);
    return pedidos.map(PedidoDTO::fromEntity);
}
```

- `Page`: devuelve el contenido de la pagina mas el total de elementos (ejecuta un `COUNT` adicional).
- `Slice`: devuelve el contenido mas un booleano indicando si hay siguiente pagina (sin `COUNT`, mas eficiente).
- `PageRequest.of(pagina, tamanio, Sort)`: crea la solicitud de pagina con ordenacion.

Para paginacion con keyset (cursor-based), mas eficiente en conjuntos grandes:

```java
@Query("SELECT p FROM Pedido p WHERE p.id > :ultimoId ORDER BY p.id")
Page<Pedido> findByIdAfter(@Param("ultimoId") Long ultimoId, Pageable pageable);
```

## Indices

Los indices de base de datos son el factor mas determinante en el rendimiento de las consultas. Sin indices, el motor debe recorrer toda la tabla secuencialmente para encontrar los datos. JPA permite definirlos mediante `@Table(indexes = ...)` o `@Index`:

```java
@Entity
@Table(name = "pedidos", indexes = {
    @Index(name = "idx_pedido_cliente", columnList = "cliente_id"),
    @Index(name = "idx_pedido_fecha_estado", columnList = "fecha, estado"),
    @Index(name = "idx_pedido_referencia", columnList = "referencia_externa", unique = true)
})
public class Pedido { ... }
```

- Indices compuestos: utiles cuando el `WHERE` filtra por multiples columnas. El orden de las columnas debe coincidir con el de la consulta.
- Indices unicos: ademas de acelerar busquedas, garantizan unicidad a nivel de BD.
- Indices funcionales (PostgreSQL): `CREATE INDEX idx_lower_email ON usuarios(lower(email))` para busquedas case-insensitive.

> Los indices no son gratuitos: ralentizan las operaciones de escritura (`INSERT`, `UPDATE`, `DELETE`) porque deben mantenerse actualizados. Hay que indexar solo las columnas que aparecen en `WHERE`, `JOIN` y `ORDER BY` de las consultas mas frecuentes.

## Connection pooling

Crear y destruir conexiones JDBC en cada peticion es extremadamente costoso. Un pool de conexiones mantiene un conjunto de conexiones abiertas y reutilizables, eliminando la sobrecarga de creacion. HikariCP es el pool por defecto en Spring Boot 2.x y 3.x, y ofrece el mejor rendimiento del ecosistema Java.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
      pool-name: HikariPool-AP
```

- `maximum-pool-size`: numero maximo de conexiones en el pool. Debe dimensionarse segun el numero de peticiones concurrentes y la latencia de la BD. Una formula orientativa es `max-pool = (num-hilos * tiempo-respuesta) / 1000`.
- `minimum-idle`: conexiones minimas inactivas que mantiene el pool.
- `connection-timeout`: tiempo maximo de espera para obtener una conexion del pool. Si se agota, se lanza `SQLTransientConnectionException`.
- `max-lifetime`: tiempo de vida maximo de una conexion. Debe ser menor que el timeout de conexion de la base de datos.

El pool debe monitorizarse con metricas de HikariCP expuestas via Actuator para detectar agotamiento de conexiones, picos de uso y tiempos de espera excesivos.
