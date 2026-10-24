# Indices: concepto y fundamentos

Un indice en MongoDB es una estructura de datos auxiliar que mantiene una referencia ordenada a un subconjunto de los campos de una coleccion, permitiendo localizar documentos sin necesidad de recorrerlos todos. Sin indice, una consulta debe examinar cada documento de la coleccion (COLLSCAN), lo que se vuelve inviable cuando la coleccion crece al orden de millones de documentos. Con indice, el motor de base de datos navega directamente a los documentos relevantes mediante una estructura arbol equilibrada (B-Tree), reduciendo el costo de las consultas de O(n) a O(log n).

## B-Tree: estructura subyacente

MongoDB utiliza arboles B+ (una variante del B-Tree) como estructura de indice predeterminada. Cada nodo del arbol contiene un rango de valores del campo indexado y punteros a los nodos hijos o a los documentos en disco. La caracteristica fundamental del B-Tree es que permanece equilibrado: todas las hojas estan a la misma profundidad, lo que garantiza un numero constante de accesos para cualquier busqueda.

```
Root: [50]
     /      \
[10, 30]    [70, 90]
```

Cuando se realiza una consulta por un campo indexado, MongoDB desciende por el arbol desde la raiz hasta la hoja correspondiente, y desde ahi recupera los documentos mediante los punteros almacenados. La insercion de un nuevo documento actualiza el arbol, dividiendo nodos cuando es necesario para mantener el equilibrio.

> El arbol B+ de WiredTiger (motor por defecto) almacena los valores del indice ordenados y los punteros a los registros en disco. Las operaciones de insercion, eliminacion y actualizacion tienen complejidad O(log n) amortizada.

## COLLSCAN vs IXSCAN

La decision entre escaneo de coleccion (COLLSCAN) y escaneo de indice (IXSCAN) es la primera que toma el planificador de consultas de MongoDB.

### COLLSCAN

El motor recorre secuencialmente todos los documentos de la coleccion, evaluando la condicion de busqueda en cada uno. Esto implica:

- Lectura completa de todos los documentos desde disco al cache de WiredTiger.
- Comparacion del filtro contra cada documento.
- Progreso inevitablemente lineal: duplicar el tamano de la coleccion duplica el tiempo de la consulta.

Para una coleccion de 10 millones de documentos, un COLLSCAN puede requerir decenas de segundos aunque el resultado final sean solo 10 documentos.

### IXSCAN

El motor navega por el arbol B-Tech del indice hasta localizar las entradas que cumplen la condicion, y luego recupera solo los documentos correspondientes. Esto implica:

- Lectura de los nodos del arbol (mucho menor que el tamano total de la coleccion).
- Acceso directo a los documentos mediante los punteros del indice.
- Tiempo de respuesta practicamente constante independientemente del tamano de la coleccion.

```javascript
// Forzar explain para ver el plan
db.usuarios.find({ email: "user@example.com" }).explain("executionStats")
// En la salida: stage: "IXSCAN" vs stage: "COLLSCAN"
```

La diferencia practica es abismal. Una consulta que tarda 15 segundos en COLLSCAN puede resolverse en 2 milisegundos con IXSCAN.

> El planificador no siempre escoge IXSCAN aunque exista un indice candidato. Si estima que el indice devolveria un porcentaje muy alto de la coleccion (tipicamente > 10-20 %), puede preferir COLLSCAN por considerarlo mas eficiente al evitar el doble acceso: primero al indice y luego al documento.

## Covered query

Una consulta cubierta (covered query) es aquella que se resuelve completamente desde el indice, sin necesidad de acceder a los documentos de la coleccion. Esto ocurre cuando:

1. Todos los campos del filtro estan en el indice.
2. Todos los campos de la proyeccion estan en el indice.
3. El indice no contiene arrays (no es multikey).

Cuando se cumple esta condicion, el plan de ejecucion muestra `stage: "IXSCAN"` y `totalDocsExamined: 0`. El indice contiene toda la informacion que necesita la consulta.

```javascript
// Dado un indice { email: 1, nombre: 1 }
db.usuarios.find(
  { email: "user@example.com" },
  { _id: 0, email: 1, nombre: 1 }
)
// Covered query: todos los campos (email, nombre) estan en el indice
// Solo se lee del indice, sin tocar los documentos
```

Para que una consulta sea cubierta, la proyeccion debe excluir explicitamente `_id` si este no forma parte del indice, ya que `_id` se incluye por defecto en toda proyeccion.

## Cuando crear indices y cuando no

Crear un indice no es una decision trivial. Cada indice acelera las lecturas pero ralentiza las escrituras y consume espacio en disco y memoria RAM.

### Escenarios donde conviene crear indice

- Campos utilizados con frecuencia en filtros de `find()`.
- Campos utilizados en operaciones de ordenacion (`sort()`).
- Campos utilizados en la primera etapa `$match` de un aggregation pipeline.
- Campos que actuan como clave foranea (referencias entre colecciones).
- Campos con alta cardinalidad (muchos valores distintos).

### Escenarios donde NO conviene crear indice

- Colecciones pequenas (menos de unos miles de documentos). El coste de mantener el indice supera el beneficio.
- Campos con baja cardinalidad (ej. un campo booleano). Un indice sobre `activo: true/false` apenas reduce el espacio de busqueda: sigue siendo la mitad de la coleccion.
- Colecciones con alta tasa de escritura y baja tasa de lectura. Cada escritura actualiza todos los indices de la coleccion.
- Indices redundantes. Si ya existe un indice compuesto `{ a: 1, b: 1 }`, crear un indice `{ a: 1 }` es innecesario (el compuesto ya cubre las consultas por `a`).

> MongoDB permite hasta 64 indices por coleccion. En la practica, mantener mas de 5-10 indices suele ser contraproducente por el impacto en escritura. Hay que medir siempre con `explain()` y `$indexStats` antes de anadir un indice nuevo.

