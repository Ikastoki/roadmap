# Optimizacion de consultas y configuracion

Una vez identificados los cuellos de botella mediante analisis y profiling, el siguiente paso es aplicar las correcciones. La optimizacion en MongoDB abarca desde la eleccion del indice correcto hasta la configuracion del motor WiredTiger, pasando por tecnicas de paginacion eficiente y el diseno de pipelines de agregacion que minimicen el volumen de datos procesados.

## Indices correctos

El indice adecuado convierte un `COLLSCAN` en un `IXSCAN` y reduce drasticamente `totalDocsExamined`. Sin embargo, crear indices sin criterio tiene un coste: cada indice adicional ralentiza las escrituras (hay que actualizar el arbol B-tree en cada insercion, actualizacion o borrado) y consume espacio en RAM y disco.

* Regla fundamental: crear indices que cubran los patrones de consulta reales. Para cada consulta lenta, analizar el filtro (`$eq`, `$gt`, `$in`, etc.) y el orden (`sort()`), y crear un indice compuesto que los englobe.
* Indices compuestos: el orden de los campos importa. Las igualdades (`$eq`) deben ir primero, seguidas de los rangos (`$gt`, `$lt`) y finalmente los campos de ordenacion (`sort()`).

```javascript
// Consulta: db.pedidos.find({ cliente: "ABC", fecha: { $gte: ISODate("2024-01-01") } }).sort({ total: -1 })
db.pedidos.createIndex({ cliente: 1, fecha: 1, total: -1 })
//                              ^ equality    ^ range        ^ sort
```

* Indices cubiertos: cuando todos los campos que necesita la consulta estan en el indice, MongoDB nunca llega a leer los documentos. `totalDocsExamined` es 0. En `explain()`, la etapa es `IXSCAN` y `FETCH` no aparece.

```javascript
// Si solo se necesita el campo "cliente" y "total", y ambos estan en el indice
db.pedidos.createIndex({ cliente: 1, total: 1 })
db.pedidos.find({ cliente: "ABC" }, { _id: 0, cliente: 1, total: 1 })
// En explain() no aparece FETCH, solo IXSCAN
```

## Proyeccion

Por defecto, `find()` devuelve el documento completo. Si la aplicacion solo necesita dos o tres campos, el ancho de banda de red y la memoria del cliente se malgastan transportando datos que nunca se usan. La proyeccion explicita limita los campos devueltos y, en combinacion con un indice cubierto, elimina la necesidad de acceder al documento.

```javascript
// Sin proyeccion: MongoDB transfiere el documento entero
db.pedidos.find({ cliente: "ABC" })

// Con proyeccion: solo devuelve los campos necesarios
db.pedidos.find({ cliente: "ABC" }, { _id: 0, total: 1, fecha: 1 })
```

> El campo `_id` se incluye por defecto. Para excluirlo hay que indicarlo explicitamente con `{ _id: 0 }`.

## Paginacion eficiente: seek method vs skip

La paginacion clasica con `skip()` y `limit()` se vuelve ineficiente en colecciones grandes porque MongoDB debe recorrer y descartar los documentos anteriores a la pagina solicitada: un `skip(100000).limit(20)` examina 100.020 documentos aunque solo devuelva 20.

```javascript
// Ineficiente: MongoDB escanea 100.020 documentos
db.pedidos.find().sort({ _id: 1 }).skip(100000).limit(20)
```

El metodo alternativo es la paginacion basada en rangos (seek method), que utiliza un filtro sobre el campo de ordenacion para posicionarse directamente donde comienza la siguiente pagina.

```javascript
// Primera pagina
var pagina = db.pedidos.find().sort({ _id: 1 }).limit(20).toArray()
var ultimoId = pagina[pagina.length - 1]._id

// Segunda pagina: usar el _id del ultimo documento como punto de partida
db.pedidos.find({ _id: { $gt: ultimoId } }).sort({ _id: 1 }).limit(20)
```

Este enfoque mantiene `totalDocsExamined` igual a `nReturned` independientemente de la profundidad de la pagina. La contrapartida es que no permite saltos arbitrarios a paginas intermedias (modelo offset-based), pero para la mayoria de casos de uso (paginacion secuencial) es drasticamente mas eficiente.

## Aggregation pipeline

El pipeline de agregacion procesa documentos en etapas. Cada etapa puede reducir, transformar o enriquecer el conjunto de datos. La clave del rendimiento es reducir el volumen lo antes posible.

### $match temprano

Colocar `$match` al principio del pipeline filtra los documentos antes de que pasen por etapas costosas como `$group`, `$lookup` o `$sort`. Si el `$match` inicial utiliza un indice, MongoDB lo aprovecha automaticamente.

```javascript
// Eficiente: $match primero reduce el conjunto antes de agrupar
db.pedidos.aggregate([
  { $match: { cliente: "ABC", fecha: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$producto", total: { $sum: "$importe" } } }
])
```

### Proyeccion temprana

`$project`, `$addFields` y `$set` deben situarse lo mas tarde posible a menos que reduzcan significativamente el tamano de los documentos. Cuando solo se necesita un subconjunto de campos, un `$project` temprano tras el `$match` reduce la memoria que utiliza cada etapa posterior.

```javascript
db.pedidos.aggregate([
  { $match: { cliente: "ABC" } },
  { $project: { _id: 0, producto: 1, importe: 1, fecha: 1 } },
  { $group: { _id: "$producto", total: { $sum: "$importe" } } }
])
```

### Evitar $unwind innecesario

`$unwind` deshace arrays y multiplica el numero de documentos. Si se usa `$unwind` seguido de `$group`, se esta duplicando trabajo. Siempre que sea posible, usar operadores que trabajen directamente sobre arrays como `$sum`, `$avg`, `$push` o `$max` en lugar de deshacer y recomponer.

```javascript
// Ineficiente: deshace 1000 documentos con arrays de 10 elementos -> 10000 documentos
db.pedidos.aggregate([
  { $unwind: "$items" },
  { $group: { _id: "$_id", total: { $sum: "$items.precio" } } }
])

// Eficiente: opera directamente sobre el array sin deshacer
db.pedidos.aggregate([
  { $addFields: { total: { $sum: "$items.precio" } } }
])
```

### Indices en $lookup

La etapa `$lookup` es un left outer join con otra coleccion. Para que sea eficiente, la coleccion foranea debe tener un indice sobre el campo que se usa en la condicion de igualdad.

```javascript
// Si se busca por "clienteId", la coleccion "clientes" debe tener indice en ese campo
db.clientes.createIndex({ clienteId: 1 })
```

## Connection pooling

Cada conexion a MongoDB consume un hilo en el servidor y recursos de red en el cliente. Abrir y cerrar conexiones continuamente es costoso. Los drivers de MongoDB implementan connection pooling por defecto, pero es necesario configurarlo adecuadamente segun la carga de trabajo.

Los parametros clave del pool de conexiones son:

- `maxPoolSize`: numero maximo de conexiones simultaneas que el driver mantiene abiertas. El valor por defecto suele ser 100. Si la aplicacion maneja peticiones concurrentes y muchas quedan en espera, aumentar este valor puede reducir la latencia. Si se supera, el driver encola las peticiones hasta que una conexion se libera.
- `minPoolSize`: numero minimo de conexiones que se mantienen abiertas, incluso si no hay actividad. Evita el coste de establecer conexiones nuevas en momentos de baja carga.
- `maxIdleTimeMS`: tiempo maximo que una conexion puede estar inactiva antes de cerrarse. Ayuda a liberar recursos en servidores con mucha rotacion de conexiones.

```javascript
// Ejemplo con Node.js driver
const client = new MongoClient(uri, {
  maxPoolSize: 50,
  minPoolSize: 5,
  maxIdleTimeMS: 30000
})
```

> Un `maxPoolSize` demasiado alto puede saturar el servidor. Cada conexion en MongoDB consume un hilo y memoria. El valor optimo depende del numero de nucleos de CPU y del patron de acceso. Una regla orientativa es `maxPoolSize = numero_de_nucleos * 10`.

## Configuracion de WiredTiger

WiredTiger es el motor de almacenamiento por defecto desde MongoDB 3.2. Su configuracion determina como se gestiona la memoria, la persistencia y la concurrencia.

### Cache de WiredTiger

El parametro `wiredTigerCacheSizeGB` controla el tamano de la cache interna de WiredTiger. Por defecto, MongoDB utiliza el 50% de la RAM total disponible menos 1 GB, con un maximo de 1 GB (esto ultimo se ajusta en versiones recientes). La cache almacena las paginas de datos e indices mas utilizadas.

- Si el working set (datos e indices a los que se accede frecuentemente) cabe en la cache, las lecturas son mayoritariamente en memoria y el rendimiento es optimo.
- Si el working set supera la cache, WiredTiger debe leer de disco constantemente (cache overflow), lo que degrada el rendimiento.

```ini
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # 8 GB de cache en un servidor con 16 GB de RAM
```

> Reducir la cache por debajo del 50% de RAM puede provocar swapping si el sistema tiene suficiente memoria libre. Aumentarla por encima del 80% puede causar competicion con el sistema operativo y otros procesos.

### Journal y commit interval

`journal.commitIntervalMs` define cada cuantos milisegundos WiredTiger sincroniza el journal (bitacora de escritura) con el disco. El valor por defecto es 100 ms.

- Reducirlo a 50 ms mejora la durabilidad (menos perdida de datos en caso de fallo) pero aumenta la presion de escritura en disco.
- Aumentarlo a 200-300 ms reduce las operaciones de E/S y puede mejorar el rendimiento en cargas de trabajo intensivas en escritura, a costa de una mayor ventana de perdida de datos.

```ini
storage:
  wiredTiger:
    engineConfig:
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```

### maxPoolSize

A nivel de servidor, `maxPoolSize` (en `net.maxIncomingConnections`) limita el numero de conexiones entrantes que MongoDB acepta. El valor por defecto es 65536 (limitado por el sistema operativo). Si se espera un numero elevado de clientes simultaneos, puede ser necesario aumentarlo.

```ini
net:
  maxIncomingConnections: 10000
```

Es importante coordinarlo con el `maxPoolSize` de los drivers: si cada driver configura un pool de 100 conexiones y hay 200 aplicaciones conectadas, el servidor recibira 20.000 conexiones potenciales.

> Cada conexion abierta consume aproximadamente 1 MB de memoria en el servidor (stack de hilo, estructuras internas). En servidores con 10.000 conexiones, solo el overhead de conexiones ronda los 10 GB de RAM.

