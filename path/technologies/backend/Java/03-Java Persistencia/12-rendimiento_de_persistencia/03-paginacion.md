# Paginacion en persistencia

La paginacion es esencial para manejar conjuntos de datos grandes sin sobrecargar la memoria de la aplicacion ni la base de datos. Aunque Spring Data JPA proporciona soporte nativo para paginacion mediante `Pageable` y `Page`, existen diferentes tecnicas para implementar paginacion a nivel de base de datos, cada una con sus ventajas y limitaciones en terminos de rendimiento y consistencia.

## LIMIT/OFFSET

La paginacion clasica mediante `LIMIT/OFFSET` es la tecnica que utiliza Spring Data JPA por defecto. Se traduce en clausulas SQL como:

```sql
SELECT * FROM usuarios ORDER BY id ASC LIMIT 20 OFFSET 40;
```

El funcionamiento es sencillo: la base de datos genera el conjunto completo de resultados (aplicando filtros y ordenamiento), descarta las primeras OFFSET filas y devuelve las siguientes LIMIT filas.

```java
Pageable pagina = PageRequest.of(2, 20, Sort.by("id").ascending());
Page<Usuario> resultado = usuarioRepository.findAll(pagina);
```

El problema fundamental de `LIMIT/OFFSET` es que el coste aumenta linealmente con el offset. Para acceder a la pagina 1000 con 20 elementos por pagina, la base de datos debe generar y descartar 20000 filas antes de devolver las 20 solicitadas. En tablas grandes, esto se traduce en tiempos de respuesta crecientes y uso innecesario de recursos.

```sql
-- Pagina 1000 con LIMIT/OFFSET
SELECT * FROM usuarios ORDER BY id LIMIT 20 OFFSET 19980;
-- La base de datos escanea y descarta 19980 filas
```

> `LIMIT/OFFSET` es adecuado para tablas con pocos registros o cuando el offset es pequeno. Para tablas con millones de filas y paginacion profunda, el rendimiento se degrada drasticamente.

## Cursor-based pagination (Seek method)

La paginacion basada en cursor (tambien conocida como seek method o keyset pagination) utiliza una condicion WHERE para desplazarse por los datos en lugar de un offset numerico. En lugar de decir "saltate las primeras N filas", se dice "devuelve las filas despues de este valor".

```sql
SELECT * FROM usuarios WHERE id > ? ORDER BY id ASC LIMIT 20;
```

Cada pagina devuelve el ultimo valor del campo de ordenacion, que se utiliza como cursor para la pagina siguiente:

```java
public List<Usuario> paginaSiguiente(Long ultimoIdVisto, int limite) {
    return usuarioRepository.findByIdGreaterThanOrderByIdAsc(ultimoIdVisto, limite);
}
```

```java
// Primera pagina
List<Usuario> pagina1 = paginaSiguiente(null, 20);
Long cursor = pagina1.get(pagina1.size() - 1).getId();

// Segunda pagina
List<Usuario> pagina2 = paginaSiguiente(cursor, 20);
```

La ventaja fundamental del cursor-based pagination es que el coste es constante independientemente de la profundidad de la pagina. Cada consulta escanea exactamente LIMIT filas, sin importar si es la pagina 1 o la pagina 10000. Ademas, los resultados son estables incluso si se insertan o eliminan filas entre consultas, porque la paginacion se basa en un valor concreto, no en una posicion.

## Keyset pagination con campos compuestos

Cuando el ordenamiento implica multiples campos, el cursor debe incluir todos ellos:

```sql
-- Orden: (fecha_creacion DESC, id DESC)
SELECT * FROM usuarios
WHERE (fecha_creacion, id) < (?, ?)
ORDER BY fecha_creacion DESC, id DESC
LIMIT 20;
```

```java
@Query("SELECT u FROM Usuario u WHERE " +
       "(u.fechaCreacion, u.id) < (:ultimaFecha, :ultimoId) " +
       "ORDER BY u.fechaCreacion DESC, u.id DESC")
List<Usuario> paginaSiguienteKeyset(
    @Param("ultimaFecha") LocalDateTime ultimaFecha,
    @Param("ultimoId") Long ultimoId,
    Pageable pageable
);
```

La sintaxis de tuplas `(col1, col2) < (val1, val2)` esta soportada en PostgreSQL, MySQL y H2, pero no en todas las bases de datos. Una alternativa portable es usar condiciones combinadas:

```java
@Query("SELECT u FROM Usuario u WHERE " +
       "(u.fechaCreacion < :ultimaFecha) OR " +
       "(u.fechaCreacion = :ultimaFecha AND u.id < :ultimoId) " +
       "ORDER BY u.fechaCreacion DESC, u.id DESC")
List<Usuario> paginaSiguienteKeysetPortable(
    @Param("ultimaFecha") LocalDateTime ultimaFecha,
    @Param("ultimoId") Long ultimoId,
    Pageable pageable
);
```

## Cursor vs Offset

La eleccion entre paginacion por cursor y por offset depende del caso de uso:

| Aspecto | LIMIT/OFFSET | Cursor |
|---------|-------------|--------|
| Rendimiento con paginacion profunda | Se degrada | Constante |
| Navegacion a pagina arbitraria | Si | No (solo siguiente/anterior) |
| Estabilidad ante cambios | Inestable | Estable |
| Complejidad de implementacion | Baja | Media |
| Compatibilidad Spring Data | Nativo | Manual |

La paginacion por cursor es ideal para scroll infinito, timelines y feeds donde el usuario siempre avanza hacia adelante. La paginacion por offset es preferible cuando se necesita acceso directo a una pagina especifica o cuando la tabla tiene pocos registros.

## Paginacion con Spring Data JPA

Spring Data JPA no soporta keyset pagination de forma nativa, pero los repositorios pueden implementarla facilmente con metodos personalizados:

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    List<Usuario> findByIdGreaterThanOrderByIdAsc(Long id, Pageable pageable);

    @Query("SELECT u FROM Usuario u WHERE u.id > :cursor ORDER BY u.id ASC")
    List<Usuario> buscarConCursor(@Param("cursor") Long cursor, Pageable pageable);
}
```

El parametro `Pageable` se utiliza solo para el tamaño de pagina (`getPageSize()`), ignorando el numero de pagina.

## Consideraciones de indices

La paginacion por cursor exige un indice que soporte la condicion de busqueda y el ordenamiento:

```sql
CREATE INDEX idx_usuarios_id ON usuarios (id);
CREATE INDEX idx_usuarios_fecha_id ON usuarios (fecha_creacion DESC, id DESC);
```

Sin el indice adecuado, la base de datos realiza un scan completo de la tabla para encontrar el punto de inicio del cursor, anulando cualquier ventaja de rendimiento.
