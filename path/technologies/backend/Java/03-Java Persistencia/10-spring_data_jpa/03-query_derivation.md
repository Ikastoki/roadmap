# Query derivation

La query derivation es el mecanismo mediante el cual Spring Data JPA genera automaticamente consultas JPQL a partir del nombre de los metodos declarados en una interfaz repository. Este sistema elimina la necesidad de escribir consultas manuales para la mayoria de los casos de uso, apoyandose en una convencion de nombres estricta y predecible que el framework parsea para construir la expresion JPQL correspondiente.

## Convencion de nombres

El nombre de un metodo derivado sigue una estructura fija. Comienza con un prefijo como `find...By`, `read...By`, `get...By`, `query...By`, `count...By` o `exists...By`. La palabra `By` marca el inicio de las clausulas de criterio. Cada criterio se compone de una propiedad de la entidad seguida opcionalmente de una palabra clave que define el operador de comparacion.

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByEmail(String email);
    List<Usuario> findByNombreAndApellido(String nombre, String apellido);
    List<Usuario> findByEdadGreaterThan(Integer edadMinima);
    List<Usuario> findByNombreContaining(String fragmento);
}
```

`findByEmail(String email)` genera la consulta `SELECT u FROM Usuario u WHERE u.email = ?1`. El framework reconoce `Email` como una propiedad de la entidad y aplica el operador de igualdad por defecto.

## Operadores soportados

Spring Data JPA soporta una amplia variedad de operadores que se expresan como palabras clave intercaladas entre la propiedad y el sufijo `By` (en el caso de multiples condiciones) o al final del nombre de la propiedad.

`And` y `Or` permiten combinar multiples condiciones en una misma consulta:

```java
List<Usuario> findByNombreAndApellido(String nombre, String apellido);
List<Usuario> findByEmailOrTelefono(String email, String telefono);
```

`Between` para rangos de valores inclusivos:

```java
List<Usuario> findByEdadBetween(Integer min, Integer max);
```

`LessThan`, `GreaterThan`, `LessThanEqual` y `GreaterThanEqual` para comparaciones numericas y de fechas:

```java
List<Usuario> findByEdadGreaterThan(Integer edadMinima);
List<Usuario> findByFechaIngresoAfter(LocalDate fecha);
List<Usuario> findByFechaIngresoBefore(LocalDate fecha);
```

`Like`, `NotLike`, `Containing`, `StartingWith`, `EndingWith` para busquedas por patrones:

```java
List<Usuario> findByNombreLike(String patron);
List<Usuario> findByNombreContaining(String fragmento);
List<Usuario> findByNombreStartingWith(String prefijo);
List<Usuario> findByNombreEndingWith(String sufijo);
```

`Containing` es equivalente a `Like %?1%` y escapa automaticamente los caracteres especiales, lo que lo hace preferible a `Like` cuando se busca un fragmento de texto.

`In` y `NotIn` para conjuntos de valores:

```java
List<Usuario> findByRolIn(List<String> roles);
```

`IsNull` y `IsNotNull` comprueban valores nulos:

```java
List<Usuario> findByEmailIsNull();
List<Usuario> findByTelefonoIsNotNull();
```

`True` y `False` para propiedades booleanas:

```java
List<Usuario> findByActivoTrue();
```

## OrderBy

La clausula `OrderBy` se anade al final del nombre del metodo para especificar el orden de los resultados. Se puede indicar la direccion con `Asc` o `Desc`:

```java
List<Usuario> findByActivoTrueOrderByFechaCreacionDesc();
List<Usuario> findByNombreContainingOrderByApellidoAscNombreAsc(String filtro);
```

> La query derivation tiene limitaciones. Consultas con mas de tres o cuatro condiciones combinadas producen nombres de metodo excesivamente largos y dificiles de leer. Para estos casos es preferible usar `@Query` con JPQL explicito.

## Ignorar mayusculas

`IgnoreCase` se puede aplicar a propiedades de tipo String para realizar comparaciones insensibles a mayusculas:

```java
List<Usuario> findByNombreIgnoreCase(String nombre);
List<Usuario> findByNombreContainingIgnoreCase(String fragmento);
```

## Distincion y limitacion

`Distinct` elimina duplicados y `First`/`Top` limita el numero de resultados:

```java
List<Usuario> findDistinctByApellido(String apellido);
Optional<Usuario> findFirstByOrderByFechaCreacionDesc();
List<Usuario> findTop5ByOrderByPuntuacionDesc();
```

## Navegacion por relaciones

Las propiedades pueden recorrer relaciones mediante el operador `_` (guion bajo) que actua como separador:

```java
List<Pedido> findByUsuario_Email(String email);
```

Este metodo busca pedidos cuyo usuario asociado tenga el email especificado. La consulta generada es `SELECT p FROM Pedido p WHERE p.usuario.email = ?1`.

## Resolucion de ambiguedad

Si una propiedad contiene palabras clave reservadas (como `And`, `Or`, `By`), se puede usar el prefijo explicito para desambiguar:

```java
List<Usuario> findByAndroideMarca(String marca);
```

En este caso, `Androide` se interpreta como una propiedad completa y no como la combinacion de `And` + `Roide`.

> Internamente, Spring Data JPA utiliza el `PartTreeJpaQuery` para parsear los nombres de los metodos. El proceso de parsing divide el nombre en partes, identifica las propiedades recorriendo el metamodelo de la entidad y construye el arbol de criterios JPQL que posteriormente se traduce a una consulta SQL.
