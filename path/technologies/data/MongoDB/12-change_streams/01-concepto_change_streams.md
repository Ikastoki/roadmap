# Change Streams

Los change streams son la puerta de entrada de MongoDB al mundo de la reactividad: permiten suscribirse a los cambios que ocurren en una coleccion, base de datos o incluso un cluster completo, y recibirlos en tiempo real conforme suceden. A diferencia de una consulta periodica (polling) que pregunta "¿hay algo nuevo?", un change stream empuja los cambios al suscriptor en el momento exacto en que ocurren. Esto es posible gracias a que MongoDB registra todas las operaciones de escritura en el oplog (operations log), un log circular que replica los cambios a los nodos secundarios del replica set. Desde MongoDB 3.6, ese mismo oplog se expone al usuario a traves de la API de change streams, tanto en replica sets como en sharded clusters. El resultado es un mecanismo de notificacion nativo, eficiente y ordenado, sin necesidad de infraestructura adicional.

## Arquitectura y funcionamiento interno

Un change stream se construye sobre el oplog del replica set. Cada operacion de escritura (insert, update, delete, replace, create index, drop collection, etc.) genera una entrada en el oplog que describe el cambio. Cuando un cliente abre un change stream, el servidor mongod establece un cursor que itera sobre las entradas del oplog a partir de un punto especifico, filtrando aquellas que coinciden con el namespace y las condiciones solicitadas.

En un sharded cluster, el mecanismo se vuelve mas complejo porque los datos estan distribuidos en multiples shards, cada uno con su propio oplog. El router mongos se encarga de abrir un change stream en cada shard y fusionar los eventos en un solo flujo ordenado globalmente. Para mantener el orden, los eventos incluyen marcas de tiempo (clusterTime) que el router utiliza para intercalar los eventos de todos los shards.

> El oplog tiene un tamano finito (por defecto, el 5% del espacio libre en disco en Linux, con un minimo de 990MB y un maximo de 50GB). Si una aplicacion no consume los eventos con la suficiente rapidez, el oplog puede sobrescribir entradas antes de que el change stream las haya leido, provocando la invalidation del cursor.

## Eventos capturados

Un change stream expone un documento JSON por cada cambio detectado. El campo `operationType` identifica la naturaleza del cambio:

- **insert**: se anadio un nuevo documento a la coleccion.
- **update**: se modificaron campos de un documento existente. Por defecto solo se devuelven los campos modificados, no el documento completo.
- **replace**: se reemplazo un documento completo mediante `replaceOne()`.
- **delete**: se elimino un documento. No incluye el contenido del documento eliminado.
- **create**: se creo una nueva coleccion.
- **drop**: se elimino una coleccion.
- **rename**: se renombro una coleccion.
- **dropDatabase**: se elimino una base de datos.
- **invalidate**: ocurre cuando el change stream deja de ser valido. Esto sucede si la coleccion se elimina, se renombra o si el oplog ha sobrescrito los eventos necesarios. Una vez recibido este evento, el change stream se cierra y no se pueden recibir mas eventos.

La estructura tipica de un evento es la siguiente:

```json
{
  "_id": { "_data": "8260F...", "resumeToken": "..." },
  "operationType": "insert",
  "clusterTime": Timestamp(1650000000, 1),
  "fullDocument": { "_id": ObjectId("..."), "campo": "valor" },
  "documentKey": { "_id": ObjectId("...") },
  "ns": { "db": "midb", "coll": "micoleccion" }
}
```

- `_id`: contiene el resume token, que identifica de forma unica la posicion del evento en el oplog.
- `clusterTime`: instante en que ocurrio la operacion segun el reloj del cluster.
- `fullDocument`: el documento completo (solo presente en inserts, replaces y updates si se especifica `fullDocument: "updateLookup"`).
- `documentKey`: identificador del documento afectado. En colecciones sharded incluye la shard key.
- `ns`: namespace que contiene la base de datos y la coleccion.

## watch() y niveles de suscripcion

La entrada a los change streams es el metodo `watch()`, que se invoca sobre una coleccion, una base de datos o un cliente de MongoDB (cluster). Cada nivel filtra los eventos segun el ambito:

```
coleccion.watch()         -> solo eventos de esa coleccion
baseDeDatos.watch()       -> todas las colecciones de la BD
cliente.watch()           -> todas las colecciones en todas las BD del cluster
```

El ambito mas restrictivo (coleccion) es el mas eficiente porque el servidor filtra antes de enviar, reduciendo la carga de red y procesamiento.

## Pipeline de agregacion en change streams

Un change stream acepta un pipeline de agregacion opcional que se aplica antes de entregar los eventos al suscriptor. Esto permite filtrar y transformar los eventos en el servidor, reduciendo el volumen de datos que viaja por la red y la logica que debe manejar el cliente.

El pipeline sigue las mismas reglas que el aggregation framework, con una limitacion importante: solo se permiten las etapas `$match` y `$project`.

```javascript
const pipeline = [
  { $match: { "operationType": "insert", "ns.coll": "pedidos" } },
  { $project: { "fullDocument": 1, "operationType": 1 } }
];

const changeStream = db.collection("ventas").watch(pipeline);
```

`$match` filtra eventos por cualquier campo del documento de cambio: `operationType`, `ns.db`, `ns.coll`, `documentKey`, o incluso el contenido del propio documento a traves de `fullDocument.campo`. `$project` limita los campos que se envian al cliente.

> En MongoDB 6.0+ se anadio soporte para `$changeStreamSplitLargeEvent`, una etapa adicional que fragmenta eventos cuyo tamano supera los 16MB, permitiendo manejar documentos que exceden el limite BSON en el contexto de un change stream.

## Resume tokens

Los resume tokens son el mecanismo que distingue a los change streams de cualquier otro sistema de colas: permiten reanudar la suscripcion exactamente desde el punto en que se interrumpio, sin perder eventos ni procesar duplicados.

Cada evento incluye un token de reanudacion en el campo `_id`. Este token es opaco para el cliente (es una cadena BSON que el servidor interpreta para localizar la posicion en el oplog). Las opciones para reanudar son dos:

- **resumeAfter**: reanuda el change stream a partir del evento especificado. El evento correspondiente al token ya fue entregado, por lo que no se recibe de nuevo. El token debe pertenecer a un evento que aun este en el oplog.
- **startAfter**: similar a resumeAfter, pero el change stream puede reanudarse incluso si el evento original ya fue eliminado del oplog. El primer evento entregado sera el siguiente al del token.

```javascript
const resumeToken = changeStream.resumeToken; // guardar tras cada evento

// ... tras una reconexion ...
const newStream = db.collection("miColec").watch([], { resumeAfter: resumeToken });
```

La principal diferencia practica es que `resumeAfter` falla si el evento referenciado ya no esta en el oplog, mientras que `startAfter` salta directamente al punto actual si el token ha expirado.

```javascript
const options = { startAfter: resumeToken };
const changeStream = db.collection("miColec").watch(pipeline, options);
```

Tambien existe `startAtOperationTime`, que permite reanudar desde un momento temporal concreto (un `Timestamp` de MongoDB), util cuando se conoce la ultima vez que el sistema estuvo operativo.

