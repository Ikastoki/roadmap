# Projections

Las proyecciones permiten recuperar únicamente los campos necesarios de una entidad en lugar de cargar el objeto completo. En una aplicación real, una entidad puede contener 20 o más columnas, pero un endpoint de lista solo necesita 3 o 4. Proyectar selectivamente reduce tanto el consumo de memoria como el tráfico entre la aplicación y la base de datos.

Spring Data ofrece dos tipos principales de proyecciones: basadas en interfaces y basadas en clases (DTOs). Ambas se definen a nivel de interfaz del repositorio y se ejecutan automáticamente sin código adicional.

## Interface-based projections

Una proyección basada en interfaz es una interfaz cuyos métodos corresponden a las propiedades que se desean proyectar. Spring Data genera automáticamente una implementación proxy en tiempo de ejecución.

```java
// Interfaz de proyeccion: solo los campos declarados
public interface ProductoBasico {
    String getNombre();
    BigDecimal getPrecio();
    String getCategoria();
}

public interface ProductoResumen {
    Long getId();
    String getNombre();

    @Value("#{target.precio * 1.21}") // Expresion SpEL
    BigDecimal getPrecioConIva();
}
```

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    // Retorna proyecciones en lugar de la entidad completa
    List<ProductoBasico> findByActivoTrue();

    ProductoBasico findByNombre(String nombre);

    <T> Collection<T> findByCategoria(String categoria, Class<T> type);
}
```

```java
// Uso
List<ProductoBasico> productos = productoRepository.findByActivoTrue();
for (ProductoBasico p : productos) {
    System.out.println(p.getNombre() + " - " + p.getPrecio());
}
```

> Las proyecciones basadas en interfaz solo funcionan con propiedades de nivel superior. No se puede acceder a relaciones asociadas o campos anidados sin recurrir a `@Value` con SpEL.

## Closed projections

Una proyección cerrada es aquella en la que todos los métodos de la interfaz corresponden directamente a propiedades de la entidad. Spring Data optimiza estas proyecciones ejecutando un `SELECT` solo con las columnas necesarias.

```java
// Proyeccion cerrada: todos los metodos mapean propiedades reales
public interface ClienteBasico {
    String getNombre();
    String getEmail();
}
```

```java
// Proyeccion abierta: al menos un metodo usa @Value
public interface ClienteInfo {
    String getNombre();

    @Value("#{target.nombre} #{target.apellido}")
    String getNombreCompleto();
}
```

Las proyecciones abiertas no pueden optimizarse a nivel de SQL porque requieren evaluar expresiones SpEL sobre la entidad completa, por lo que siempre cargan el objeto completo de la base de datos.

## Class-based projections (DTOs)

Las proyecciones basadas en clases utilizan clases POJO o registros como目标 de la proyección. Requieren un constructor que acepte los mismos parámetros que las propiedades proyectadas.

```java
public class ProductoDTO {
    private final String nombre;
    private final BigDecimal precio;
    private final String categoria;

    public ProductoDTO(String nombre, BigDecimal precio, String categoria) {
        this.nombre = nombre;
        this.precio = precio;
        this.categoria = categoria;
    }

    // Getters sin setters (objeto inmutable)
    public String getNombre() { return nombre; }
    public BigDecimal getPrecio() { return precio; }
    public String getCategoria() { return categoria; }
}

// Alternativa: record (Java 16+)
public record ProductoRecord(String nombre, BigDecimal precio, String categoria) {}
```

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    @Query("SELECT new com.ejemplo.dto.ProductoDTO(p.nombre, p.precio, p.categoria) " +
           "FROM Producto p WHERE p.activo = true")
    List<ProductoDTO> productosActivosDTO();

    @Query("SELECT new com.ejemplo.dto.ProductoDTO(p.nombre, p.precio, p.categoria) " +
           "FROM Producto p WHERE p.categoria = :categoria")
    List<ProductoDTO> productosPorCategoria(@Param("categoria") String categoria);
}
```

La sentencia `new` dentro de la consulta JPQL es el mecanismo que permite a JPA instanciar directamente el DTO sin pasar por la entidad. Esto es significativamente más eficiente que proyectar a la entidad y luego transformar a DTO manualmente.

## Dynamic projections

Las proyecciones dinámicas permiten al llamador decidir en tiempo de ejecución qué tipo de proyección desea, utilizando un parámetro de tipo genérico.

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    <T> Collection<T> findByCategoria(String categoria, Class<T> type);

    <T> Collection<T> findByActivoTrue(Class<T> type);
}
```

```java
// El cliente decide la proyeccion en tiempo de ejecucion
@Service
public class ProductoService {

    public Collection<ProductoBasico> obtenerBasico(String categoria) {
        return productoRepository.findByCategoria(categoria, ProductoBasico.class);
    }

    public Collection<ProductoDTO> obtenerDTO(String categoria) {
        return productoRepository.findByCategoria(categoria, ProductoDTO.class);
    }
}
```

```java
// En el controller
@GetMapping("/api/productos/basico")
public Collection<ProductoBasico> listarBasico(@RequestParam String categoria) {
    return productoService.obtenerBasico(categoria);
}

@GetMapping("/api/productos/completo")
public Collection<ProductoDTO> listarDTO(@RequestParam String categoria) {
    return productoService.obtenerDTO(categoria);
}
```

> Las proyecciones dinámicas son especialmente útiles en APIs REST donde diferentes clientes (web, móvil, interno) pueden requerir diferentes niveles de detalle de los mismos recursos.

