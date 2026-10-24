# Paginacion y sorting

La paginacion y el ordenamiento son funcionalidades esenciales en cualquier aplicacion que maneje conjuntos de datos potencialmente grandes. Spring Data JPA proporciona un soporte nativo y elegante para ambas operaciones a traves de las interfaces `Pageable`, `Sort`, `Page` y `Slice`, permitiendo implementar desde paginacion clasica con numeros de pagina hasta tecnicas de scroll infinito sin necesidad de escribir codigo SQL manual.

## La interfaz Pageable

`Pageable` es la interfaz central para la paginacion en Spring Data. Encapsula la informacion necesaria para ejecutar una consulta paginada: el numero de pagina (comenzando en 0), el tamaño de pagina y los criterios de ordenamiento.

```java
Pageable primeraPagina = PageRequest.of(0, 10);
Pageable segundaPagina = PageRequest.of(1, 10);
```

`PageRequest` es la implementacion mas utilizada de `Pageable`. Su metodo estatico `of()` acepta el indice de pagina y el tamaño. Opcionalmente, se puede anadir un objeto `Sort`:

```java
Pageable paginaOrdenada = PageRequest.of(0, 10, Sort.by("apellido").ascending());
```

Los metodos del repository que aceptan `Pageable` devuelven `Page<T>`:

```java
Page<Usuario> pagina = usuarioRepository.findAll(paginacion);
```

## La clase Sort

`Sort` encapsula uno o varios criterios de ordenamiento. Se construye mediante una API fluida que permite combinar multiples campos con diferentes direcciones:

```java
Sort orden = Sort.by("apellido").ascending()
                 .and(Sort.by("nombre").descending());
```

`Sort.by()` acepta el nombre de la propiedad y opcionalmente un `Direction`:

```java
Sort orden = Sort.by(Direction.DESC, "fechaCreacion");
Sort ordenMultiple = Sort.by(Direction.ASC, "apellido", "nombre");
```

> `Sort` y `Pageable` utilizan los nombres de las propiedades de la entidad, no los nombres de las columnas de la base de datos. Spring Data JPA se encarga de la traduccion al nombre de columna mediante el metamodelo de JPA.

## La interfaz Page<T>

`Page<T>` extiende `Slice<T>` y proporciona informacion completa sobre la pagina actual y los totales globales:

```java
Page<Usuario> pagina = usuarioRepository.findAll(paginacion);

List<Usuario> contenido = pagina.getContent();
int numeroPagina = pagina.getNumber();
int tamanoPagina = pagina.getSize();
int totalPaginas = pagina.getTotalPages();
long totalElementos = pagina.getTotalElements();
boolean tieneAnterior = pagina.hasPrevious();
boolean tieneSiguiente = pagina.hasNext();
```

El metodo `getTotalElements()` ejecuta una consulta `SELECT COUNT(*)` adicional a la consulta principal. Esta operacion tiene un coste en bases de datos con millones de registros, especialmente si la consulta implica JOINs multiples o filtros complejos.

`getTotalPages()` calcula el numero de paginas dividiendo el total de elementos entre el tamaño de pagina, redondeando hacia arriba.

`getContent()` devuelve la lista de entidades correspondientes a la pagina actual. El tamaño de esta lista puede ser menor que `getSize()` si la pagina es la ultima.

`hasNext()` y `hasPrevious()` permiten determinar si existen paginas adyacentes sin necesidad de inspeccionar los totales.

## La interfaz Slice<T>

`Slice<T>` es una alternativa ligera a `Page<T>`. Solo garantiza que sepa si existe una pagina siguiente, pero no proporciona acceso al total de elementos ni al numero total de paginas.

```java
Slice<Usuario> slice = usuarioRepository.findByActivoTrue(paginacion);
boolean hayMas = slice.hasNext();
List<Usuario> contenido = slice.getContent();
```

Para que un metodo devuelva `Slice<T>`, debe declararse explicitamente con ese tipo de retorno:

```java
Slice<Usuario> findByActivoTrue(Pageable pageable);
```

`Slice<T>` es ideal para implementaciones de scroll infinito en aplicaciones web, donde solo se necesita saber si hay mas datos disponibles sin necesidad de mostrar el numero exacto de paginas. Como no ejecuta la consulta `COUNT(*)`, su rendimiento es superior al de `Page<T>` en conjuntos de datos grandes.

## Paginacion desde el controlador

En aplicaciones Spring MVC, los parametros de paginacion pueden vincularse automaticamente desde la peticion HTTP:

```java
@GetMapping("/usuarios")
public Page<Usuario> listarUsuarios(Pageable pageable) {
    return usuarioRepository.findAll(pageable);
}
```

Spring MVC extrae los parametros `page`, `size` y `sort` de la query string (`?page=0&size=10&sort=apellido,asc`) y construye automaticamente el objeto `Pageable`. La configuracion de valores por defecto se realiza en `application.properties`:

```properties
spring.data.web.pageable.default-page-size=20
spring.data.web.pageable.max-page-size=2000
spring.data.web.sort.sort-parameter=sort
```

## Estrategias de paginacion

La paginacion clasica basada en `LIMIT/OFFSET` que utiliza Spring Data JPA por defecto tiene una limitacion conocida: a medida que el offset aumenta, el rendimiento se degrada porque la base de datos debe escanear y descartar las filas anteriores. Para tablas muy grandes, existen alternativas como la paginacion por cursor (keyset pagination), aunque Spring Data JPA no la soporta de forma nativa y requiere implementacion manual.

> `Page<T>` ejecuta dos consultas: una para los datos y otra para el count. En tablas con millones de registros y filtros complejos, `Slice<T>` puede ser una alternativa mas eficiente si no se necesita el total exacto de elementos.

## Combinacion con @Query

Los metodos anotados con `@Query` tambien soportan `Pageable` y `Sort`. Spring Data JPA modifica automaticamente la consulta JPQL para incorporar las clausulas de paginacion y ordenamiento:

```java
@Query("SELECT u FROM Usuario u WHERE u.activo = true AND u.edad >= :edadMinima")
Page<Usuario> buscarActivosMayoresDe(@Param("edadMinima") Integer edad, Pageable pageable);
```

La consulta count que `Page<T>` necesita se genera automaticamente a partir de la consulta JPQL original. No obstante, se puede proporcionar una consulta count personalizada si la generada automaticamente no es optima:

```java
@Query(value = "SELECT u FROM Usuario u WHERE u.activo = true",
       countQuery = "SELECT COUNT(u) FROM Usuario u WHERE u.activo = true")
Page<Usuario> buscarActivos(Pageable pageable);
```
