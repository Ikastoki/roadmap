# Projections

Las proyecciones permiten extraer subconjuntos especificos de datos de una entidad, en lugar de cargar la entidad completa con todas sus propiedades y relaciones. Spring Data JPA soporta multiples mecanismos de proyeccion que van desde interfaces simples hasta DTOs con constructores personalizados, ofreciendo flexibilidad para adaptar la cantidad de datos recuperados a las necesidades especificas de cada caso de uso.

## Interface-based projections

Las proyecciones basadas en interfaces definen los campos que se desean recuperar mediante metodos getter. Spring Data JPA genera automaticamente implementaciones proxy en tiempo de ejecucion:

```java
public interface UsuarioResumen {
    String getNombre();
    String getEmail();
}
```

El repository declara metodos que devuelven la interfaz de proyeccion:

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    List<UsuarioResumen> findByActivoTrue();
}
```

Spring Data JPA genera una consulta que solo selecciona las columnas correspondientes a los metodos definidos en la interfaz: `SELECT u.nombre, u.email FROM Usuario u WHERE u.activo = true`.

Las proyecciones pueden incluir propiedades anidadas mediante navegacion:

```java
public interface PedidoResumen {
    Long getId();
    String getDescripcion();
    UsuarioResumen getUsuario();

    interface UsuarioResumen {
        String getNombre();
    }
}
```

Esta proyeccion recupera el ID y la descripcion del pedido junto con el nombre del usuario asociado. Spring Data JPA genera la consulta optimizada con los JOINs necesarios.

## Closed projections

Cuando todos los getters de la interfaz corresponden directamente a propiedades de la entidad, se denomina closed projection. Spring Data JPA puede optimizar la consulta seleccionando unicamente las columnas necesarias, lo que reduce la cantidad de datos transferidos desde la base de datos.

```java
public interface UsuarioNombreEmail {
    String getNombre();
    String getEmail();
}
```

## Open projections

Las open projections utilizan la anotacion `@Value` con expresiones SpEL para calcular valores que no existen directamente en la entidad:

```java
public interface UsuarioNombreCompleto {
    @Value("#{target.nombre + ' ' + target.apellido}")
    String getNombreCompleto();
}
```

`target` hace referencia a la entidad subyacente. En una open projection, Spring Data JPA no puede optimizar la consulta porque necesita cargar todas las propiedades referenciadas en las expresiones SpEL para evaluarlas en memoria.

```java
public interface UsuarioDetalle {
    String getNombre();
    String getApellido();

    @Value("#{target.nombre + ' ' + target.apellido}")
    String getNombreCompleto();
}
```

En este caso, aunque solo se exportan `nombre` y `apellido` explicitamente, la expresion SpEL en `getNombreCompleto()` obliga a cargar ambas propiedades.

> Las open projections con `@Value` cargan la entidad completa. Si el objetivo es reducir la carga de datos, las closed projections son significativamente mas eficientes.

## Class-based projections (DTOs)

Las proyecciones tambien pueden implementarse como clases DTO (Data Transfer Object) con un constructor que recibe los campos necesarios. Este enfoque requiere un constructor parametrizado en el DTO:

```java
public class UsuarioDTO {
    private final String nombre;
    private final String email;

    public UsuarioDTO(String nombre, String email) {
        this.nombre = nombre;
        this.email = email;
    }

    public String getNombre() { return nombre; }
    public String getEmail() { return email; }
}
```

El repository se declara con la proyeccion DTO:

```java
@Query("SELECT new com.ejemplo.dto.UsuarioDTO(u.nombre, u.email) FROM Usuario u")
List<UsuarioDTO> buscarTodosDTO();
```

A diferencia de las proyecciones por interfaz, los DTOs requieren una consulta JPQL explicita con la expresion `new`. No funcionan con query derivation.

## Dynamic projections

Spring Data JPA permite que un mismo metodo del repository acepte la clase de proyeccion como parametro, delegando al invocador la decision de que proyeccion utilizar:

```java
<T> List<T> findByActivoTrue(Class<T> tipoProyeccion);
```

El invocador decide en el momento de la llamada:

```java
List<UsuarioResumen> resumen = usuarioRepository.findByActivoTrue(UsuarioResumen.class);
List<UsuarioDTO> dto = usuarioRepository.findByActivoTrue(UsuarioDTO.class);
```

Este patron es util cuando el mismo caso de uso necesita diferentes niveles de detalle segun el contexto. El repository mantiene una unica declaracion de metodo y los clientes eligen la proyeccion adecuada.

## Proyecciones con JPQL nativo

En consultas nativas, se puede mapear el resultado directamente a una interfaz de proyeccion:

```java
@Query(value = "SELECT u.nombre, u.email FROM usuarios u WHERE u.activo = true",
       nativeQuery = true)
List<UsuarioResumen> buscarActivosNativo();
```

Spring Data JPA convierte automaticamente cada fila del resultado en un proxy que implementa la interfaz de proyeccion.

## Ventajas de las proyecciones

Las proyecciones reducen la cantidad de datos transferidos entre la base de datos y la aplicacion. En entidades con muchas columnas o relaciones LAZY, evitan la carga innecesaria de datos que nunca se utilizan. Ademas, al seleccionar solo las columnas necesarias, el motor de base de datos puede utilizar indices de cubrimiento (covering indexes), mejorando significativamente el rendimiento de las consultas.

Para colecciones, se puede utilizar `@Projection` en combinacion con el modulo de QueryDSL o Criteria API, aunque el soporte nativo de Spring Data JPA para proyecciones en colecciones es limitado y a menudo requiere consultas `@Query` explicitas.
