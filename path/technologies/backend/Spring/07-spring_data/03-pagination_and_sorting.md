# Pagination and Sorting

La paginación y el ordenación son mecanismos esenciales para manejar conjuntos de datos de forma eficiente en aplicaciones con grandes volúmenes de información. Spring Data abstrae estos conceptos mediante las interfaces `Pageable` y `Sort`, proporcionando una capa uniforme que se adapta a diferentes motores de base de datos sin modificar el código de la aplicación.

Cargar millones de registros en memoria para presentar una lista paginada de 20 elementos es un desperdicio de recursos que puede colapsar tanto la JVM como la base de datos. La paginación traslada la responsabilidad de delimitar el resultado al motor de almacenamiento, ejecutando únicamente la porción de datos que el cliente necesita en cada momento.

## Pageable

`Pageable` es una interfaz que encapsula toda la información necesaria para construir una consulta paginada: número de página, tamaño de página y opcionalmente un criterio de ordenación. La implementación más utilizada es `PageRequest`.

```java
// Pagina 0, tamaño 10, ordenados por nombre ascendente
Pageable pageable = PageRequest.of(0, 10, Sort.by("nombre").ascending());

// Sin ordenacion
Pageable pageable = PageRequest.of(2, 20);

// Con multiples criterios de ordenacion
Sort sort = Sort.by(
    Sort.Order.desc("fechaCreacion"),
    Sort.Order.asc("nombre")
);
Pageable pageable = PageRequest.of(0, 15, sort);
```

El número de página es 0-indexed, es decir, la primera página es la página 0. Esto se debe a que internamente se calcula el `OFFSET` como `pageNumber * pageSize`, por lo que empezar en 0 garantiza que la primera consulta no omita registros.

```java
@Repository
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    Page<Pedido> findByEstado(EstadoPedido estado, Pageable pageable);

    Page<Pedido> findByClienteId(Long clienteId, Pageable pageable);
}

// Uso
@Service
public class PedidoService {
    private final PedidoRepository pedidoRepository;

    public Page<Pedido> obtenerPedidosPendientes(int pagina, int tamanio) {
        Pageable pageable = PageRequest.of(pagina, tamanio, Sort.by("fechaCreacion").descending());
        return pedidoRepository.findByEstado(EstadoPedido.PENDIENTE, pageable);
    }
}
```

## Page\<T\>

`Page<T>` es el resultado de una consulta paginada. Contiene no solo los datos de la página actual, sino también metadatos completos sobre el conjunto total de resultados.

```java
Page<Pedido> pagina = pedidoRepository.findByEstado(EstadoPedido.PENDIENTE, pageable);

// Metadatos disponibles
int numeroPagina = pagina.getNumber();        // Pagina actual (0-indexed)
int tamanioPagina = pagina.getSize();         // Tamaño de pagina
int totalPaginas = pagina.getTotalPages();    // Total de paginas
long totalElementos = pagina.getTotalElements(); // Total de registros
boolean primeraPagina = pagina.isFirst();
boolean ultimaPagina = pagina.isLast();
boolean vacia = pagina.isEmpty();

// Datos
List<Pedido> pedidos = pagina.getContent();   // Lista de elementos
```

> `Page<T>` realiza una consulta adicional con `COUNT` para determinar el total de registros. Este coste extra es necesario para calcular el total de páginas, pero puede ser evitado utilizando `Slice<T>` cuando no se necesita esa información.

```java
@GetMapping("/api/pedidos")
public ResponseEntity<Map<String, Object>> obtenerPedidos(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {

    Page<Pedido> resultado = pedidoService.obtenerPedidosPendientes(page, size);

    Map<String, Object> response = new HashMap<>();
    response.put("pedidos", resultado.getContent());
    response.put("paginaActual", resultado.getNumber());
    response.put("totalPaginas", resultado.getTotalPages());
    response.put("totalElementos", resultado.getTotalElements());

    return ResponseEntity.ok(response);
}
```

## Slice\<T\>

`Slice<T>` es una alternativa ligera a `Page<T>` que no realiza la consulta COUNT. En su lugar, carga un elemento adicional para determinar si existen más registros en la siguiente página. Esto lo convierte en la opción ideal para implementar scroll infinito donde el usuario nunca necesita conocer el total exacto de registros.

```java
Slice<Pedido> slice = pedidoRepository.findByEstado(EstadoPedido.PENDIENTE, pageable);

boolean tieneSiguiente = slice.hasNext();    // true si hay mas elementos
List<Pedido> pedidos = slice.getContent();
```

La diferencia fundamental es que `Slice` solo necesita ejecutar la consulta principal con `LIMIT (size + 1)`, mientras que `Page` ejecuta tanto la consulta principal como un `COUNT` separado. Para tablas con millones de registros, esta diferencia puede ser significativa.

```java
@Repository
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    Slice<Producto> findByCategoria(String categoria, Pageable pageable);
}

// Controlador para scroll infinito
@GetMapping("/api/productos/scroll")
public Slice<Producto> productosScroll(
        @RequestParam String categoria,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "30") int size) {

    Pageable pageable = PageRequest.of(page, size);
    return productoRepository.findByCategoria(categoria, pageable);
}
```

## Sort

`Sort` encapsula uno o más criterios de ordenación que se aplican directamente en la consulta SQL. Se puede construir de forma estática o programática.

```java
// Ordenacion simple
Sort sort = Sort.by("nombre").ascending();

// Ordenacion descendente
Sort sort = Sort.by("fechaCreacion").descending();

// Multiples criterios
Sort sort = Sort.by(
    Sort.Order.desc("prioridad"),
    Sort.Order.asc("fechaCreacion")
);

// OrdenacionIgnoreCase
Sort sort = Sort.by(Sort.Order.by("nombre").ignoreCase());
```

El uso de `Direction` permite construir el orden dinámicamente según parámetros del cliente, algo habitual en APIs REST.

```java
Sort sort = Sort.by(Direction.ASC, "nombre");
Sort sortDesc = Sort.by(Direction.DESC, "precio");
```

## Pageable por defecto en controller

Spring MVC permite inyectar `Pageable` directamente como parámetro de método en un controller, configurando un valor por defecto para evitar que el cliente envíe parámetros inválidos.

```java
@RestController
@RequestMapping("/api/productos")
public class ProductoController {

    private final ProductoRepository productoRepository;

    @GetMapping
    public Page<Producto> listar(
            @PageableDefault(size = 20, sort = "nombre",
                           direction = Sort.Direction.ASC) Pageable pageable) {
        return productoRepository.findAll(pageable);
    }
}
```

```yaml
# application.yml
spring:
  data:
    web:
      pageable:
        default-page-size: 20
        max-page-size: 100
        one-indexed-parameters: false
```

> Cuando `one-indexed-parameters` es `true`, el parámetro `page` empieza en 1 en lugar de 0, lo que puede ser útil para clientes que esperan una numeración humana. Sin embargo, internamente Spring Data sigue usando 0-indexed.

```java
// Paginacion en multiples endpoints del mismo controller
@RestController
@RequestMapping("/api")
public class ReporteController {

    @GetMapping("/ventas")
    public Page<Venta> ventas(
            @PageableDefault(size = 10) Pageable pageable,
            @RequestParam(required = false) String categoria) {
        if (categoria != null) {
            return ventaRepository.findByCategoria(categoria, pageable);
        }
        return ventaRepository.findAll(pageable);
    }

    @GetMapping("/inventario")
    public Page<Producto> inventario(
            @PageableDefault(size = 25, sort = "stock",
                           direction = Sort.Direction.DESC) Pageable pageable) {
        return productoRepository.findAll(pageable);
    }
}
```

La paginación en Spring Data es transparente tanto en la capa de repositorio como en la de presentación. El motor de base de datos se encarga de ejecutar las consultas `LIMIT/OFFSET` o su equivalente, y la aplicación recibe objetos `Page` o `Slice` con toda la información necesaria para construir la respuesta.

