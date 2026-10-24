# Cache en Hibernate

Hibernate incorpora un sistema de cache en multiples niveles cuyo objetivo es reducir el numero de consultas a la base de datos y mejorar el rendimiento general de la aplicacion. Este sistema se compone de tres niveles diferenciados: el cache de primer nivel (first-level cache), el cache de segundo nivel (second-level cache) y el cache de consultas (query cache). Cada uno tiene un alcance, una duracion y un comportamiento distinto.

## First-level cache

El cache de primer nivel esta asociado a la sesion de Hibernate (`Session`). Cuando se carga una entidad mediante `get()`, `load()`, `find()` o cualquier consulta, Hibernate almacena la entidad en este cache antes de devolverla. Si durante la misma sesion se vuelve a solicitar la misma entidad (por el mismo identificador), Hibernate la devuelve desde el cache sin ejecutar ninguna consulta SQL.

```java
// Primera carga: ejecuta SELECT
Usuario u1 = session.get(Usuario.class, 1L);

// Segunda carga: no ejecuta SELECT, devuelve la misma instancia
Usuario u2 = session.get(Usuario.class, 1L);

assert u1 == u2; // Es el mismo objeto en memoria
```

El first-level cache es obligatorio y no se puede desactivar. Su principal beneficio es evitar lecturas repetitivas dentro de una misma transaccion, pero tiene una consecuencia importante: si no se gestiona adecuadamente, puede consumir mucha memoria.

> Cuando se procesan lotes de datos, el first-level cache debe limpiarse periodicamente llamando a `session.clear()` para evitar que la sesion acumule miles de entidades y agote la memoria disponible.

```java
for (int i = 0; i < 10000; i++) {
    Proceso p = new Proceso();
    p.setNombre("Proceso " + i);
    session.persist(p);

    if (i % 50 == 0) {
        session.flush();
        session.clear();
    }
}
```

## Second-level cache

El cache de segundo nivel es un cache compartido entre sesiones. Las entidades almacenadas en este cache son visibles desde cualquier sesion que pertenezca al mismo `SessionFactory`. A diferencia del first-level cache, el second-level cache es opcional y debe configurarse explicitamente.

El second-level cache requiere un proveedor externo. Los mas utilizados son:

- **Ehcache**: El mas extendido en proyectos legacy. Ligero, maduro y facil de configurar.
- **Infinispan**: Desarrollado por Red Hat (la misma compania detras de Hibernate). Soporta clustering y distribucion.
- **Hazelcast**: Enfocado en entornos distribuidos, con replicacion y particionado de datos.
- **Redis**: Mediante implementaciones personalizadas o modulos de terceros.

### Configuracion

Se habilita en `hibernate.cfg.xml` o en `application.properties`:

```xml
<property name="hibernate.cache.use_second_level_cache">true</property>
<property name="hibernate.cache.region.factory_class">
    org.hibernate.cache.ehcache.EhCacheRegionFactory
</property>
```

Y se marca cada entidad como cacheable con `@Cacheable` y `@Cache`:

```java
@Entity
@Table(name = "categorias")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Categoria {
    @Id
    private Long id;
    private String nombre;
}
```

### Estrategias de concurrencia

Cada entidad puede definir la estrategia de concurrencia para el second-level cache:

- **READ_ONLY**: Para entidades que nunca se modifican (datos de referencia, catalogos). Maximum rendimiento.
- **READ_WRITE**: Para entidades que se leen y escriben frecuentemente. Usa un mecanismo de bloqueo optimista (soft-lock) para garantizar consistencia.
- **NONSTRICT_READ_WRITE**: Similar a READ_WRITE pero sin garantia de consistencia estricta. Adecuado cuando es aceptable leer datos ligeramente desactualizados.
- **TRANSACTIONAL**: Para entornos con soporte de transacciones JTA. Ofrece aislamiento completo pero requiere un proveedor de cache transaccional.

> La estrategia READ_WRITE no es adecuada para entornos clustering sin un proveedor de cache que soporte sincronizacion distribuida. Si varias instancias de la aplicacion modifican la misma entidad en sus caches locales sin coordinacion, pueden producirse inconsistencias.

### Comportamiento

Cuando el second-level cache esta habilitado para una entidad, la carga a traves de `get()` o `load()` sigue esta secuencia:

1. Busca la entidad en el first-level cache (sesion actual).
2. Si no esta, busca en el second-level cache (compartido).
3. Si no esta, ejecuta la consulta SQL y almacena el resultado en ambos caches.

Para las consultas por criterios de busqueda (no por identificador), el second-level cache no se utiliza a menos que se combine con el cache de consultas.

## Query cache

El cache de consultas almacena los resultados de consultas HQL/JPQL y Criteria API. A diferencia del second-level cache, que indexa por identificador de entidad, el query cache almacena los IDs de las entidades que devuelve una consulta concreta.

```xml
<property name="hibernate.cache.use_query_cache">true</property>
```

```java
List<Producto> productos = session.createQuery(
    "FROM Producto p WHERE p.categoria = :cat", Producto.class)
    .setParameter("cat", "ELECTRONICA")
    .setCacheable(true)  // Habilita query cache para esta consulta
    .list();
```

> El query cache solo almacena los IDs de las entidades, no las entidades completas. Cuando se ejecuta una consulta cacheada, Hibernate recupera los IDs del query cache y luego busca cada entidad por su ID, primero en el first-level cache, luego en el second-level cache y finalmente en la base de datos. Por lo tanto, el query cache solo es efectivo cuando el second-level cache tambien esta habilitado.

### Cache de regiones

Tanto el second-level cache como el query cache organizan los datos en regiones. Cada entidad tiene su propia region y las consultas tambien pueden tener regiones especificas. Esto permite configurar politicas de expiracion, tamanio maximo y estrategia de desalojo diferentes para cada region.

```java
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "catalogos")
@Entity
public class Pais {
    // ...
}
```

La gestion manual de las regiones se realiza mediante `SessionFactory.getCache()`:

```java
session.getSessionFactory().getCache()
    .evictEntityRegion(Pais.class);  // Invalida toda la region de Pais
session.getSessionFactory().getCache()
    .evictQueryRegion("consultas.lentas");
session.getSessionFactory().getCache()
    .evictAll();
```

> El cache de segundo nivel y el cache de consultas son herramientas potentes pero no son una solucion universal. Antes de habilitarlos es recomendable medir el impacto real con herramientas de profiling. Un cache mal configurado puede provocar lecturas sucias, estancamiento de memoria y comportamientos inconsistentes en entornos clusterizados.
