# Gestion de entidades

El EntityManager de JPA no es un simple ejecutor de SQL: es un gestor de estado que mantiene un contexto de persistencia donde las entidades viven, cambian y eventualmente se sincronizan con la base de datos. Comprender el ciclo de vida de las entidades y las operaciones que lo gobiernan es esencial para escribir codigo de persistencia predecible y evitar comportamientos inesperados como datos duplicados, cambios no persistidos o excepciones de detach.

## Ciclo de vida de las entidades

Toda entidad JPA atraviesa cuatro estados a lo largo de su existencia. El EntityManager es el unico responsable de gestionar las transiciones entre estos estados.

```
                  persist()
    New ───────────────────────> Managed
     ^                              │
     │                              │ find(), JPQL...
     │                              │
     │         merge()              │ flush() / commit()
     │<─────────────────────        v
     │                    ────   Base de datos
     │                  /
     │    Detached <───
     │                remove()
     │    (antes Managed)
     │
     └─── remove():
    Removed (sincronizado en flush())
```

### New (o Transient)

Una entidad recien creada con `new` no tiene representacion en la base de datos ni esta asociada a ningun EntityManager. El ORM la ignora completamente. Su estado es "transitorio": si no se persiste antes de que la variable salga de ambito, la entidad se pierde sin dejar rastro.

```java
Producto producto = new Producto();
producto.setNombre("Teclado mecanico");
// producto esta en estado NEW - no existe en BD ni en el contexto de persistencia
```

### Managed (o Persistent)

Cuando una entidad pasa a estado managed, el EntityManager la rastrea: cualquier cambio en sus atributos se detecta automaticamente y se sincroniza con la base de datos al hacer `flush()`. Una entidad pasa a managed mediante:

- `em.persist(entity)`: la entidad pasa de new a managed.
- `em.find(Class, id)`: la entidad cargada desde BD se devuelve en estado managed.
- `em.merge(entity)`: una entidad detached se "refusiona" y la copia devuelta esta en estado managed.
- Consultas JPQL/Criteria: todas las entidades devueltas en resultados estan en estado managed.

```java
Producto producto = new Producto("Teclado mecanico");
em.persist(producto);           // NEW -> MANAGED
producto.setPrecio(BigDecimal.valueOf(89.90));  // Se rastrea automaticamente
// Al hacer commit(), se genera: UPDATE productos SET precio = 89.90 WHERE id = ...
```

En estado managed, el EntityManager mantiene una copia de los valores originales de la entidad. Al hacer `flush()`, compara los valores actuales con los originales y solo genera UPDATE para las columnas que cambiaron (gracias a la estrategia de dirty checking).

### Detached

Una entidad detached es aquella que fue managed pero cuyo EntityManager se ha cerrado o contra la que se ha ejecutado `em.detach(entity)`. Tambien entran en este estado las entidades serializadas y enviadas a traves de la red (por ejemplo, en una API REST).

```java
em.getTransaction().begin();
Producto producto = em.find(Producto.class, 1L);
em.getTransaction().commit();
// producto ahora esta DETACHED: el EntityManager se cerro o la transaccion termino

producto.setPrecio(BigDecimal.valueOf(99.00));
// Este cambio NO se persistira automaticamente
em.merge(producto);  // necesario para volver a managed
```

Una entidad detached se reconoce porque su `@Id` no es null pero el EntityManager no la conoce. Intentar modificar una entidad detached fuera de una transaccion y esperar que se persista es uno de los errores mas frecuentes.

### Removed

Una entidad en estado removed esta marcada para eliminacion. El registro se eliminara de la base de datos en el proximo `flush()`.

```java
Producto producto = em.find(Producto.class, 1L);
em.remove(producto);    // MANAGED -> REMOVED
em.flush();             // Se ejecuta DELETE FROM productos WHERE id = 1
```

Despues de `flush()`, la entidad pasa a estado new (su identificador se mantiene pero ya no esta asociada al contexto de persistencia). Llamar a `remove()` sobre una entidad que no esta managed lanza `IllegalArgumentException`.

## Operaciones principales

### persist

Transiciona una entidad de new a managed. Si la entidad ya tiene un `@Id` generado por la base de datos, el INSERT puede diferirse hasta el `flush()`. Si la entidad usa `GenerationType.IDENTITY`, el INSERT se ejecuta inmediatamente para obtener el ID generado.

```java
em.persist(producto);
// Si producto tiene IDENTITY, aqui ya se ejecuta INSERT
// Si producto tiene SEQUENCE, solo asigna un ID de la secuencia y difiere el INSERT
```

`persist()` no retorna nada. La entidad pasada como argumento se modifica in-place para reflejar su nuevo estado managed.

### merge

Toma una entidad detached y devuelve una copia managed de la misma. Si la entidad no existe en la base de datos, la inserta. Si existe, actualiza los valores.

```java
// Escenario tipico: entidad recibida desde un formulario web
Producto productoDTO = new Producto();
productoDTO.setId(1L);
productoDTO.setNombre("Teclado inalambrico");
productoDTO.setPrecio(BigDecimal.valueOf(120.00));

// productoDTO esta DETACHED (tiene ID pero no esta en ningun EM)
Producto managed = em.merge(productoDTO);
```

`merge()` funciona en cinco pasos:
1. Comprueba si existe una entidad managed con el mismo ID en el contexto de persistencia.
2. Si existe, copia los valores de la entidad detached a la managed.
3. Si no existe, la carga de la base de datos (o crea una nueva si no hay registro).
4. Copia los valores y devuelve la entidad managed.
5. La entidad original (detached) sigue detached.

> `merge()` devuelve una entidad diferente a la que se pasa. Es un error comun seguir usando la entidad original pensando que es managed.

### find

Busca una entidad por su clave primaria. Devuelve la entidad en estado managed o null si no existe. Es la operacion de lectura mas simple y eficiente porque puede servirse de la cache de primer nivel sin ejecutar SQL.

```java
Producto producto = em.find(Producto.class, 1L);
// SQL: SELECT * FROM productos WHERE id = 1 (solo si no esta en cache)

Producto producto2 = em.find(Producto.class, 1L);
// Sin SQL: devuelve la misma instancia de la cache de primer nivel

Producto inexistente = em.find(Producto.class, 999L);
// null
```

La diferencia con `getReference()` es sutil pero importante: `getReference()` devuelve un proxy sin cargar los datos reales hasta que se accede a ellos. Si la entidad no existe, `getReference()` lanza `EntityNotFoundException` al acceder al proxy, no al invocar el metodo.

### remove

Marca una entidad managed para eliminacion. La entidad debe estar en estado managed; si no lo esta, lanza `IllegalArgumentException`. Tras `remove()`, la entidad pasa a estado removed y se elimina al hacer `flush()`.

```java
em.getTransaction().begin();
Producto producto = em.find(Producto.class, 1L);
em.remove(producto);
em.getTransaction().commit();
// Se ejecuta: DELETE FROM productos WHERE id = 1
```

Si la entidad tiene relaciones con `cascade = CascadeType.REMOVE` o `cascade = CascadeType.ALL`, eliminar el padre elimina en cascada a los hijos.

### refresh

Recarga el estado de una entidad desde la base de datos, descartando cualquier cambio local no persistido. Es util cuando se sospecha que otra transaccion ha modificado los datos subyacentes.

```java
Producto producto = em.find(Producto.class, 1L);
producto.setNombre("Cambio local");

em.refresh(producto);
// El cambio local se descarta y producto vuelve al estado de la BD
// SQL: SELECT * FROM productos WHERE id = 1
```

`refresh()` solo funciona sobre entidades managed. Fuera de la transaccion o despues de cerrar el EntityManager, lanza `IllegalArgumentException`.

### contains

Comprueba si una entidad esta en estado managed dentro del contexto de persistencia actual. Devuelve `true` si la entidad esta siendo rastreada por el EntityManager.

```java
Producto producto = em.find(Producto.class, 1L);
em.contains(producto);  // true

em.detach(producto);
em.contains(producto);  // false

Producto nuevo = new Producto();
nuevo.setId(2L);
em.contains(nuevo);     // false (aunque tenga ID, no esta en el EM)
```

`contains()` es util en pruebas y depuracion para verificar el estado de una entidad, especialmente en escenarios donde el ciclo de vida se vuelve complejo debido a cascadas y detached.
