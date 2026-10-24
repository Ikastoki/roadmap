# Repositories

El patron Repository abstrae el acceso a datos proporcionando una interfaz orientada al dominio, ocultando los detalles de la tecnologia de persistencia subyacente. Spring Data JPA automatiza gran parte de esta capa generando implementaciones en tiempo de ejecucion a partir de interfaces, pero su uso incorrecto puede difuminar la separacion de responsabilidades y convertir los repositorios en contenedores de logica de negocio dispersa.

## Interfaces limpias

Un repositorio debe exponer unicamente operaciones que tengan sentido en el lenguaje del dominio. No es necesario ni deseable heredar todos los metodos de `JpaRepository` o `CrudRepository` si no se usan. Muchos proyectos empiezan exponiendo `findAll()`, `save()`, `deleteById()` y terminan con metodos que no tienen significado de negocio.

```java
// MAL: expone operaciones genericas sin contexto de dominio
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
}

// BIEN: define solo las operaciones que necesita el dominio
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {

    Optional<Usuario> findByEmail(String email);

    boolean existsByEmail(String email);

    @Query("SELECT u FROM Usuario u WHERE u.activo = true ORDER BY u.nombre")
    List<Usuario> findAllActivos();

    long countByActivoTrue();
}
```

Para repositorios que solo necesitan un subconjunto de operaciones, se puede crear una interfaz base personalizada:

```java
@NoRepositoryBean
public interface ReadOnlyRepository<T, ID> extends Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    long count();
}

public interface ProductoRepository extends ReadOnlyRepository<Producto, Long> {
    Optional<Producto> findBySku(String sku);
}
```

## No logica de negocio en repos

El repositorio tiene una unica responsabilidad: traducir peticiones de datos a consultas y devolver resultados. Cualquier logica que implique decisiones de negocio, calculos, validaciones o transformaciones complejas debe residir en la capa de servicio.

```java
// MAL: logica de negocio en el repositorio
@Repository
public class PedidoRepositoryImpl implements PedidoRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    public void aplicarDescuentoAFecha(Long clienteId, LocalDate fechaLimite) {
        List<Pedido> pedidos = em.createQuery(
            "SELECT p FROM Pedido p WHERE p.cliente.id = :clienteId AND p.fecha < :fechaLimite",
            Pedido.class)
            .setParameter("clienteId", clienteId)
            .setParameter("fechaLimite", fechaLimite)
            .getResultList();

        for (Pedido p : pedidos) {
            p.setDescuentoAplicado(true);
            p.setTotal(p.getTotal().multiply(BigDecimal.valueOf(0.9)));
        }
    }
}
```

La logica de descuento debe estar en un servicio que use el repositorio solo para recuperar los datos y para persistir:

```java
@Service
@Transactional
public class PromocionService {

    private final PedidoRepository pedidoRepository;
    private final DescuentoCalculator descuentoCalculator;

    public void aplicarDescuentoAFecha(Long clienteId, LocalDate fechaLimite) {
        List<Pedido> pedidos = pedidoRepository.findByClienteIdAndFechaBefore(clienteId, fechaLimite);
        pedidos.forEach(p -> {
            BigDecimal descuento = descuentoCalculator.calcular(p.getTotal());
            p.setDescuentoAplicado(true);
            p.setTotal(descuento);
        });
        pedidoRepository.saveAll(pedidos);
    }
}
```

## Usar Spring Data

Spring Data JPA implementa los metodos de consulta derivados del nombre del metodo mediante analisis de la firma. Para la mayoria de las consultas simples no es necesario escribir JPQL manualmente. El motor de consultas soporta prefijos como `findBy`, `existsBy`, `countBy`, `deleteBy`, seguidos de criterios de filtro.

```java
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    List<Pedido> findByClienteId(Long clienteId);

    List<Pedido> findByEstadoAndFechaBetween(String estado, LocalDate inicio, LocalDate fin);

    List<Pedido> findByTotalGreaterThanEqual(BigDecimal totalMinimo);

    Optional<Pedido> findTopByClienteIdOrderByFechaDesc(Long clienteId);

    boolean existsByReferenciaExterna(String referencia);

    void deleteByClienteIdAndEstado(Long clienteId, String estado);
}
```

El analizador de consultas soporta `And`, `Or`, `Between`, `LessThan`, `GreaterThan`, `Like`, `StartingWith`, `EndingWith`, `Containing`, `OrderBy`, `Top`, `First`, `Distinct`, ademas de anidar propiedades mediante `_` para evitar ambiguedades (ej. `findByCliente_Nombre`).

## @Query para consultas complejas

Cuando la logica de la consulta supera la capacidad de los metodos derivados, se usa `@Query` con JPQL o SQL nativo. Esto mantiene la consulta dentro de la interfaz del repositorio, visible y documentada, en lugar de oculta en implementaciones.

```java
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    @Query("""
        SELECT p FROM Pedido p
        JOIN FETCH p.cliente
        JOIN FETCH p.lineas
        WHERE p.estado = :estado
        ORDER BY p.fecha DESC
        """)
    List<Pedido> findByEstadoConDetalles(@Param("estado") String estado);

    @Query("""
        SELECT new com.empresa.dto.ResumenPedido(
            p.id, c.nombre, p.total, p.fecha
        )
        FROM Pedido p
        JOIN p.cliente c
        WHERE p.fecha BETWEEN :inicio AND :fin
        """)
    List<ResumenPedido> resumenPorPeriodo(@Param("inicio") LocalDate inicio,
                                          @Param("fin") LocalDate fin);

    @Query(value = """
        SELECT EXTRACT(MONTH FROM p.fecha) as mes,
               SUM(p.total) as total
        FROM pedidos p
        WHERE p.fecha >= :desde
        GROUP BY EXTRACT(MONTH FROM p.fecha)
        ORDER BY mes
        """, nativeQuery = true)
    List<Object[]> totalesPorMes(@Param("desde") LocalDate desde);
}
```

> Las consultas nativas (`nativeQuery = true`) son portables al motor de BD especifico. Se recomienda usarlas solo cuando JPQL no pueda expresar la consulta (funciones ventana, operaciones especificas del motor). En esos casos, conviene aislarlas en repositorios separados para facilitar la migracion de BD.

## Transacciones en servicio, no en repositorio

Los metodos de `JpaRepository` heredan el comportamiento transaccional de la implementacion base (tipicamente `SimpleJpaRepository`), que ya esta anotado con `@Transactional(readOnly = true)` para los metodos de lectura y `@Transactional` para escritura. Sin embargo, la capa de servicio debe ser la que defina los limites transaccionales, agrupando multiples operaciones de repositorio en una misma transaccion.

```java
@Service
@Transactional(readOnly = true)
public class PedidoService {

    private final PedidoRepository pedidoRepository;
    private final NotificacionService notificacionService;

    public Optional<Pedido> buscarPorId(Long id) {
        return pedidoRepository.findById(id);
    }

    @Transactional
    public Pedido crearPedido(CrearPedidoRequest request) {
        Pedido pedido = new Pedido(request.getClienteId());
        pedido.addLineas(request.getLineas());
        Pedido guardado = pedidoRepository.save(pedido);
        notificacionService.notificarPedidoCreado(guardado);
        return guardado;
    }

    @Transactional
    public void cancelarPedido(Long id) {
        Pedido pedido = pedidoRepository.findById(id)
            .orElseThrow(() -> new PedidoNoEncontradoException(id));
        pedido.cancelar();
        // No es necesario llamar a save(): el dirty checking
        // de Hibernate persiste los cambios automaticamente
    }
}
```

Los repositorios deben ser agnosticos respecto al contexto transaccional en el que se ejecutan. Definir `@Transactional` en el repositorio puede provocar transacciones anidadas inesperadas o impedir que el servicio agrupe varias operaciones. La unica excepcion son los metodos batch o los repositorios personalizados que requieren un contexto transaccional explicito.
