# Rendimiento

El rendimiento en MongoDB no es el resultado de una configuracion magica, sino de la aplicacion sistematica de practicas de diseno, indexacion y operacion. Los indices adecuados, las consultas optimizadas y una configuracion de conexion correcta marcan la diferencia entre un sistema que responde en milisegundos y uno que colapsa bajo carga.

## Indices en campos de consulta

Un indice es la estructura mas efectiva para acelerar consultas. Sin indice, MongoDB realiza un collection scan: lee todos los documentos de la coleccion y evalua la condicion en cada uno. Con el indice adecuado, la busqueda se reduce a un recorrido del arbol B+ que contiene solo los valores indexados y las referencias a los documentos.

No todos los campos necesitan un indice. La regla es indexar los campos que aparecen en las condiciones de filtro (`$match`, `find`), ordenacion (`$sort`) y en las etapas de `$lookup`. Un indice compuesto sobre `{ estado: 1, fecha_creacion: -1 }` cubre a la vez un filtro por estado y una ordenacion por fecha.

Los indices tienen coste: ralentizan las escrituras (cada insercion o actualizacion debe actualizar el arbol B+) y consumen memoria y disco. El indice ideal es el que maximiza la cobertura con el menor numero de campos.

```javascript
// Indice efectivo para: db.pedidos.find({ estado: "activo" }).sort({ fecha: -1 })
db.pedidos.createIndex({ estado: 1, fecha: -1 })

// Indice poco util: cubre una consulta que rara vez se ejecuta
db.pedidos.createIndex({ color_favorito: 1 })
```

> Usar `db.collection.explain("executionStats")` para verificar que la consulta utiliza el indice esperado y examinar el numero de documentos examinados frente a los devueltos. La relacion ideal es 1:1.

## Covered queries

Una covered query ocurre cuando todos los campos que necesita la consulta estan contenidos en el propio indice, sin necesidad de acceder al documento original. MongoDB lee directamente del arbol B+ del indice, lo que reduce drasticamente el acceso a disco.

Para lograr una covered query, el indice debe incluir todos los campos de filtro, ordenacion y proyeccion. La proyeccion explicita es necesaria: MongoDB solo lee del indice si la proyeccion especifica exclusivamente campos indexados y excluye expresamente `_id` (salvo que `_id` tambien este en el indice).

```javascript
// Indice compuesto
db.productos.createIndex({ categoria: 1, precio: 1, nombre: 1 })

// Covered query: todos los campos estan en el indice y _id esta excluido
db.productos.find(
   { categoria: "electronica" },
   { _id: 0, precio: 1, nombre: 1 }
).sort({ precio: -1 })
```

El plan de ejecucion de una covered query muestra `IXSCAN` (index scan) sin `FETCH` adicional. Verificar esto con `explain()` confirma que la optimizacion es efectiva.

## Proyeccion

Enviar documentos completos por la red cuando solo se necesitan tres campos es un desperdicio de ancho de banda, memoria en el servidor y tiempo de serializacion/deserializacion. La proyeccion limita los campos que MongoDB devuelve al cliente.

```javascript
// Ineficiente: devuelve el documento completo
db.usuarios.find({ email: "user@example.com" })

// Eficiente: solo devuelve nombre y rol
db.usuarios.find(
   { email: "user@example.com" },
   { _id: 0, nombre: 1, rol: 1 }
)
```

En el aggregation framework, la proyeccion se realiza con `$project` y debe colocarse lo antes posible en el pipeline. Un `$project` temprano reduce el tamano de los documentos que las etapas posteriores deben procesar en memoria.

## Aggregation eficiente

El aggregation framework es potente, pero mal utilizado puede consumir cantidades desorbitadas de memoria y CPU. Las dos reglas de oro son: filtrar pronto y evitar operaciones costosas innecesariamente.

`$match` debe ser la primera etapa del pipeline siempre que sea posible. Si el `$match` filtra por un campo indexado, MongoDB lo utiliza como si fuera un `find()`, reduciendo el conjunto de documentos antes de cualquier transformacion. Un `$match` tardio obliga a que todas las etapas anteriores procesen documentos que seran descartados al final.

`$unwind` es una de las etapas mas costosas porque descompone cada documento en tantos documentos como elementos tenga el array. Si el array tiene 1000 elementos, un solo documento genera 1000. Conviene usar `$unwind` solo cuando es estrictamente necesario y, si es posible, sustituirlo por `$reduce` o `$map` combinados con `$project`.

```javascript
// Costoso: descompone un array de 500 elementos para filtrar uno
db.pedidos.aggregate([
   { $unwind: "$items" },
   { $match: { "items.sku": "ABC123" } }
])

// Eficiente: filtra con $elemMatch sin descomponer
db.pedidos.find({ "items.sku": "ABC123" })
```

Otras etapas que requieren atencion:

- `$sort` en memoria: si el pipeline contiene `$sort` sin un indice que lo respalde, MongoDB ordena en memoria con un limite de 100 MB. Superado ese limite, la operacion falla a menos que se habilite `allowDiskUse`.
- `$group`: consume memoria proporcional al numero de grupos distintos. Si hay millones de grupos, el pipeline puede agotar la RAM.
- `$lookup`: realiza una operacion de busqueda en otra coleccion. Sin un indice en el campo de la coleccion externa, el rendimiento se degrada gravemente.

## Connection pooling

Cada conexion a MongoDB consume un hilo en el servidor (en el pool de hilos de WiredTiger) y recursos en el cliente. Abrir y cerrar conexiones por cada operacion es prohibitivo. El connection pooling mantiene un conjunto de conexiones reutilizables que se toman y devuelven segun la demanda.

El tamano del pool depende de la concurrencia esperada. Un valor tipico esta entre 100 y 500 conexiones. Por debajo de 100, las aplicaciones con picos de trafico pueden quedarse sin conexiones disponibles y encolar peticiones. Por encima de 500, el servidor dedica recursos excesivos a gestionar hilos en lugar de procesar datos.

```javascript
// Ejemplo con driver Node.js
const client = new MongoClient(uri, {
   maxPoolSize: 200,
   minPoolSize: 50,
   maxIdleTimeMS: 30000,
   waitQueueTimeoutMS: 5000
})
```

La configuracion tipica para un servidor con 8-16 vCPUs y 32-64 GB de RAM es un pool de 200-300 conexiones. Para servidores mas potentes, el pool puede escalar hasta 500. El monitoreo de `currentActive` y `currentInUse` en las metricas del driver ayuda a ajustar este valor.

## Batch operations

Ejecutar operaciones una a una en un bucle es la forma mas ineficiente de interactuar con MongoDB. Cada operacion individual implica una ida y vuelta a la base de datos (round trip) con su latencia de red y overhead de serializacion.

`insertMany` permite insertar multiples documentos en una sola operacion. El comportamiento por defecto es una unida escritura en el servidor, mucho mas eficiente que N inserciones individuales.

```javascript
// Ineficiente: N round trips
for (const doc of documentos) {
   await db.coleccion.insertOne(doc)
}

// Eficiente: 1 round trip
await db.coleccion.insertMany(documentos)
```

Para operaciones mixtas (inserciones, actualizaciones, eliminaciones), `bulkWrite` ofrece la maxima flexibilidad con el minimo numero de viajes:

```javascript
await db.pedidos.bulkWrite([
   { insertOne: { document: { _id: 1, total: 100 } } },
   { updateOne: { filter: { _id: 2 }, update: { $set: { total: 200 } } } },
   { deleteOne:  { filter: { _id: 3 } } }
])
```

`bulkWrite` admite dos modos de ordenacion: `ordered: true` (por defecto), donde las operaciones se ejecutan secuencialmente y se detienen ante el primer error, y `ordered: false`, donde MongoDB ejecuta las operaciones en paralelo y continua aunque algunas fallen. El modo desordenado es mas rapido pero no garantiza el orden de ejecucion.

> El tamano de cada lote debe limitarse a unos pocos miles de operaciones para evitar consumir demasiada memoria en el servidor. Un lote de 5000 documentos insertados es un buen punto de partida.

