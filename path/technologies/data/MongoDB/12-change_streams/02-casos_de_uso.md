# Casos de uso de Change Streams

Los change streams convierten a MongoDB en una fuente de eventos en tiempo real, eliminando la necesidad de pollings constantes o de middleware adicional para detectar cambios. Esto abre un abanico de aplicaciones que van desde la sincronizacion de datos entre sistemas hasta la auditoria detallada de operaciones, pasando por la integracion con plataformas de streaming de eventos como Kafka. En todos los casos, la clave es que el cambio se detecta y se propaga en el mismo momento en que ocurre, no en el siguiente ciclo de polling.

## Sincronizacion de datos entre sistemas

Uno de los patrones mas habituales consiste en mantener una cache o un sistema secundario sincronizado con MongoDB en tiempo real. Cuando un documento se inserta, actualiza o elimina en la base de datos, el change stream lo detecta y dispara la accion correspondiente en el sistema destino.

```javascript
const changeStream = db.collection("usuarios").watch([{ $match: { "operationType": "update" } }]);

changeStream.on("change", (event) => {
  const key = event.documentKey._id;
  const nuevoValor = event.fullDocument;
  redisClient.set(`usuario:${key}`, JSON.stringify(nuevoValor));
});
```

Este ejemplo actualiza una cache en Redis cada vez que un documento en la coleccion `usuarios` se modifica. El resultado es que la cache siempre refleja el estado actual de la base de datos sin necesidad de TTLs ni estrategias de invalidacion complejas. El mismo patron se aplica para sincronizar datos con Elasticsearch: cada cambio en MongoDB dispara una indexacion en el motor de busqueda, manteniendo ambos sistemas alineados.

En entornos multi-base de datos, un change stream puede servir como puente entre MongoDB y un almacen relacional. Por ejemplo, cuando se crea un pedido en MongoDB, el change stream detecta el insert y replica los datos esenciales en PostgreSQL para los modulos que requieren consultas SQL complejas.

## Notificaciones en tiempo real

Las aplicaciones que necesitan notificar a los usuarios sobre cambios en los datos se benefician directamente de los change streams. En lugar de que el cliente periodicamente consulte si hay novedades, el servidor empuja la notificacion en el momento exacto.

```javascript
// Backend que escucha cambios y los reenvia por WebSocket
const changeStream = db.collection("mensajes").watch([{ $match: { "operationType": "insert" } }]);

changeStream.on("change", (event) => {
  const mensaje = event.fullDocument;
  wss.clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(mensaje));
    }
  });
});
```

Este patron es la base de sistemas de chat, feeds de actividad en redes sociales, paneles de monitoreo en vivo y notificaciones de alertas. La latencia entre la escritura en MongoDB y la llegada al cliente es del orden de milisegundos, limitada unicamente por la red.

> En aplicaciones con alta concurrencia, es recomendable usar un patron de `throttling` o `debouncing` en el lado del servidor para no saturar a los clientes cuando ocurren multiples cambios en un intervalo muy corto.

## Auditoria de cambios

Registrar quien cambio que y cuando es un requisito comun en sistemas financieros, sanitarios y cualquier aplicacion con requerimientos de compliance. Los change streams permiten construir un log de auditoria sin modificar la logica de la aplicacion principal.

```javascript
const pipeline = [
  { $match: { "operationType": { $in: ["insert", "update", "delete"] } } },
  { $addFields: { "audit.fecha": "$$NOW", "audit.origen": "cambio directo" } }
];

const changeStream = db.collection("cuentas").watch(pipeline);

changeStream.on("change", (event) => {
  db.collection("auditoria_cuentas").insertOne({
    tipo: event.operationType,
    documento: event.documentKey,
    datos: event.fullDocument || null,
    timestamp: event.clusterTime
  });
});
```

El registro de auditoria se almacena en una coleccion separada (`auditoria_cuentas`), inmmutable y con indices adecuados para consultas forenses. Dado que el change steam captura el cambio en el momento exacto, el log de auditoria es completo y no depende de que la aplicacion recuerde registrar cada operacion.

Para cumplir con regulaciones como SOX o HIPAA, es posible combinar el change stream con el audit logging nativo de MongoDB, obteniendo dos capas de registro: una operativa para desarrollo y otra de sistema para cumplimiento normativo.

## Integracion con Kafka

La combinacion de change streams con Apache Kafka es probablemente la integracion mas potente del ecosistema MongoDB. MongoDB actua como fuente de eventos (source) y Kafka como bus de eventos distribuida que alimenta multiples consumidores.

El patron tipico utiliza un conector oficial: el MongoDB Kafka Sink/Source Connector. El Source Connector abre un change stream en una o varias colecciones y publica cada cambio en un topico de Kafka.

```
MongoDB -> Change Stream -> Kafka (topico "mongo.usuarios") -> Consumidores
```

Los consumidores pueden ser:

- **Elasticsearch**: indexar documentos para busqueda full-text.
- **Spark**: ejecutar analiticas en tiempo real sobre los cambios.
- **Microservicios**: reaccionar a cambios en el sistema origen sin acoplamiento directo.
- **Data warehouses**: replicar datos a Snowflake, BigQuery o Redshift.

```json
// Evento publicado por el conector en Kafka
{
  "topic": "mongo.miBD.miColeccion",
  "key": "{ \"_id\": \"...\" }",
  "value": {
    "op": "c",   // c=create, u=update, d=delete, r=read
    "ns": "miBD.miColeccion",
    "after": { "_id": "...", "campo": "valor" },
    "before": null
  }
}
```

> El conector oficial de MongoDB para Kafka soporta exactly-once semantics y manejo automatico de resume tokens, lo que garantiza que ningun evento se pierde aunque el conector se reinicie.

## Indexacion en Elasticsearch

Mantener Elasticsearch sincronizado con MongoDB es un caso de uso clasico resuelto elegantemente con change streams. Cada insercion, actualizacion o eliminacion en MongoDB se refleja automaticamente en el indice de Elasticsearch, eliminando la necesidad de jobs periodicos de reindexacion.

El flujo puede ser directo (change stream -> Elasticsearch) o indirecto a traves de Kafka (change stream -> Kafka -> Logstash/Elasticsearch). La ruta indirecta es preferible cuando existen multiples consumidores de los mismos eventos o cuando se necesita procesamiento intermedio (transformaciones, enriquecimiento, filtrado).

```javascript
const { Client } = require("@elastic/elasticsearch");
const esClient = new Client({ node: "http://localhost:9200" });

const changeStream = db.collection("productos").watch();

changeStream.on("change", async (event) => {
  if (event.operationType === "delete") {
    await esClient.delete({ index: "productos", id: event.documentKey._id.toString() });
  } else {
    await esClient.index({ index: "productos", id: event.documentKey._id.toString(), body: event.fullDocument });
  }
});
```

## Propagacion de eventos en microservicios

En una arquitectura de microservicios, los change streams pueden actuar como mecanismo de comunicacion interno sin necesidad de un bus de eventos externo. Un microservicio expone sus cambios a traves de un change stream que otros microservicios consumen directamente.

Este enfoque es particularmente util en dominios donde los microservicios ya comparten la base de datos MongoDB o donde el acoplamiento via base de datos compartida es aceptable. Sin embargo, para sistemas con requisitos de escalabilidad mas exigentes, Kafka sigue siendo la opcion recomendada por su capacidad de almacenar eventos de forma duradera y permitir replay.

## Cache distribuida e invalidacion inteligente

Los change streams permiten implementar estrategias de invalidacion de cache basadas en eventos reales, superando las limitaciones de los TTL fijos. Cuando un documento se modifica en MongoDB, el change stream notifica al sistema de cache que debe invalidar o actualizar la entrada correspondiente, no antes ni despues, sino en el instante preciso.

```javascript
// Servicio de cache que escucha cambios en productos
const changeStream = db.collection("productos").watch([{ $match: { operationType: "update" } }]);

changeStream.on("change", (event) => {
  const cacheKey = `producto:${event.documentKey._id}`;
  cache.del(cacheKey); // Invalida cache; la proxima lectura recargara desde MongoDB
});
```

