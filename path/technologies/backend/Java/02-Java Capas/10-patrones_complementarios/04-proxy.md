# Proxy

El patron Proxy proporciona un sustituto o representante de otro objeto para controlar el acceso a este. El proxy implementa la misma interfaz que el objeto real y delega las llamadas tras ejecutar logica adicional: retardar la creacion del objeto real, verificar permisos, cachear resultados o registrar accesos. El cliente no distingue entre el proxy y el objeto real, lo que permite anadir comportamientos transversales sin modificar el codigo existente.

## Lazy loading con FetchType.LAZY

El caso de uso mas extendido de Proxy en aplicaciones Java con JPA es la carga perezosa de asociaciones. Cuando una entidad tiene una relacion `@ManyToOne` o `@OneToMany`, el proveedor JPA puede configurarla como `FetchType.LAZY` para no traer los datos asociados hasta que se acceda a ellos.

```java
@Entity
public class Pedido {
    @Id
    private Long id;

    @OneToMany(mappedBy = "pedido", fetch = FetchType.LAZY)
    private List<LineaPedido> lineas;
}
```

Hibernate no almacena directamente una coleccion vacia o los datos reales en el campo `lineas`, sino que coloca un proxy --una subclase sintetica de la clase de la coleccion-- que intercepta la primera llamada al iterador. En ese momento, si el EntityManager aun esta abierto, Hibernate ejecuta la consulta SQL necesaria y carga los datos. El mismo mecanismo aplica a las entidades proxy para asociaciones `@ManyToOne`: Hibernate genera una subclase por bytecode enhancement o por defecto mediante la creacion de un objeto que solo tiene el identificador y que carga el resto de atributos al primer acceso.

```java
Pedido pedido = em.find(Pedido.class, 1L);
// En este punto lineas es un proxy (PersistentBag), no se ha lanzado SQL

List<LineaPedido> lineas = pedido.getLineas();
// Al invocar iterator() o size() se ejecuta: SELECT * FROM lineas_pedido WHERE pedido_id = 1
```

Este comportamiento es transparente para el codigo de negocio, pero exige que el contexto de persistencia (EntityManager) este abierto en el momento del acceso. Si el proxy se accede fuera de una transaccion o despues de cerrar el EntityManager, se produce la temida excepcion `LazyInitializationException`. Por esta razon es habitual disenar los servicios para que carguen explicitamente las asociaciones necesarias mediante `JOIN FETCH` en la consulta JPQL o mediante `EntityGraph`, en lugar de confiar en que el proxy estara disponible en la capa de presentacion.

```java
public Pedido findWithLineas(Long pedidoId) {
    return em.createQuery(
        "SELECT p FROM Pedido p JOIN FETCH p.lineas WHERE p.id = :id",
        Pedido.class)
        .setParameter("id", pedidoId)
        .getSingleResult();
}
```

## Cache con proxy

Un proxy puede actuar como cache: ante una llamada repetida a un metodo costoso, el proxy devuelve el valor previamente calculado sin invocar al objeto real. Esta variante se conoce como caching proxy o virtual proxy.

```java
public interface CatalogoProductos {
    Producto obtenerPorId(Long id);
}

public class CatalogoProductosCacheProxy implements CatalogoProductos {

    private final CatalogoProductos real;
    private final Map<Long, Producto> cache = new ConcurrentHashMap<>();

    public CatalogoProductosCacheProxy(CatalogoProductos real) {
        this.real = real;
    }

    @Override
    public Producto obtenerPorId(Long id) {
        return cache.computeIfAbsent(id, real::obtenerPorId);
    }
}
```

El cliente inyecta el proxy en lugar del servicio real. El proxy decide si delegar o no, y el cliente no tiene conocimiento de la politica de cache.

## Proxy de seguridad o control de acceso

Otra variante es el protection proxy, que verifica permisos antes de delegar la llamada. Un objeto que realiza operaciones sensibles (borrar pedidos, modificar roles) puede protegerse detras de un proxy que comprueba el contexto de seguridad.

```java
public class PedidoServiceSecurityProxy implements PedidoService {

    private final PedidoService real;
    private final SecurityContext securityContext;

    @Override
    public void anularPedido(Long pedidoId) {
        if (!securityContext.hasRole("ADMIN")) {
            throw new AccesoDenegadoException();
        }
        real.anularPedido(pedidoId);
    }
}
```

## Proxy remoto

El proxy remoto representa un objeto que reside en un espacio de direcciones diferente --otro proceso, otro servidor-- y se encarga de serializar los parametros, transmitirlos por la red y deserializar la respuesta. Es la base de tecnologias como RMI, EJB remotos o clientes HTTP que implementan una interfaz local que oculta la comunicacion HTTP subyacente (por ejemplo, FeignClient en Spring Cloud).

```java
@FeignClient(name = "servicio-pagos", url = "https://api.pagos.com")
public interface PagoProxy {
    @PostMapping("/cobrar")
    ResultadoPago cobrar(@RequestBody CobroRequest request);
}
```

El `PagoProxy` es un proxy generado dinamicamente: el cliente llama a un metodo Java normal y el proxy construye la peticion HTTP, la envia y mapea la respuesta.

## Proxy estatico frente a proxy dinamico

Los ejemplos anteriores son proxies estaticos: una clase escrita a mano que implementa la misma interfaz que el sujeto real. Java tambien permite proxies dinamicos mediante `java.lang.reflect.Proxy`, que crea una implementacion de interfaz en tiempo de ejecucion. Este mecanismo es la base de aspectos como `@Transactional` en Spring: el contenedor crea un proxy dinamico que intercepta la llamada, abre una transaccion, invoca el metodo real y confirma o deshace segun el resultado.

```java
PedidoService proxy = (PedidoService) Proxy.newProxyInstance(
    PedidoService.class.getClassLoader(),
    new Class<?>[] { PedidoService.class },
    (obj, method, args) -> {
        // Logica antes de la invocacion
        Object resultado = method.invoke(servicioReal, args);
        // Logica despues de la invocacion
        return resultado;
    });
```

Frameworks como Spring prefieren usar proxies basados en CGLIB o el `Proxy` de Java segun si la clase objetivo implementa interfaces o no. En cualquier caso, el patron subyacente es siempre el mismo: un intermediario que controla el acceso al objeto real sin que el cliente lo perciba.
