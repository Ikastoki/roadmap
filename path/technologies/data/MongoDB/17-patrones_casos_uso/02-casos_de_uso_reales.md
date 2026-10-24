# Casos de Uso Reales con MongoDB

MongoDB se utiliza en produccion en miles de organizaciones que abarcan desde startups hasta empresas del Fortune 500. Cada caso de uso impone requisitos distintos sobre el modelo de datos, el patron de acceso y la infraestructura. Analizar estos escenarios reales permite entender como se combinan los patrones de diseno y las funcionalidades de MongoDB para resolver problemas concretos.

## Catalogo de Productos

El catalogo de productos es el caso de uso fundacional de MongoDB en el comercio electronico. La variabilidad de atributos entre categorias (una camiseta tiene talla y color; un portatil tiene RAM y procesador) hace que el modelo relacional requiera tablas EAV (Entity-Attribute-Value) o multiples tablas con joins, mientras que MongoDB lo resuelve de forma nativa con el patron Attribute.

Un documento tipico de producto combina esquema flexible, datos anidados y referencias:

```json
{
  "_id": ObjectId("..."),
  "sku": "LAP-001",
  "nombre": "Laptop Ultrabook 15",
  "precio": 1299.99,
  "categoria_id": ObjectId("..."),
  "marca": "TechBrand",
  "atributos": [
    { "k": "ram", "v": "32GB" },
    { "k": "procesador", "v": "Intel Core i7-13700H" },
    { "k": "almacenamiento", "v": "1TB SSD NVMe" },
    { "k": "pantalla", "v": "15.6 pulgadas 4K" }
  ],
  "variantes": [
    { "color": "Plata", "sku": "LAP-001-S", "stock": 25 },
    { "color": "Negro", "sku": "LAP-001-N", "stock": 10 }
  ],
  "valoraciones": {
    "promedio": 4.5,
    "total": 234
  },
  "fecha_creacion": ISODate("2026-01-15T10:00:00Z")
}
```

Atlas Search permite anadir busqueda de texto completa con soporte para facetas, sugerencias y busqueda por vectores sin necesidad de un motor externo como Elasticsearch. Los indices de busqueda se definen sobre los campos relevantes (nombre, descripcion, atributos) y las consultas se realizan mediante el aggregation framework usando el operador `$search`.

```javascript
db.productos.aggregate([
  {
    $search: {
      compound: {
        must: [{ text: { query: "ultrabook 4k", path: "nombre" } }],
        filter: [{ range: { path: "precio", gte: 500, lte: 2000 } }]
      }
    }
  },
  { $limit: 20 }
])
```

La combinacion de esquema flexible, atributos variables y busqueda integrada convierte a MongoDB en una plataforma unica para catalogos, eliminando la complejidad operativa de mantener multiples sistemas.

## Comentarios y Foros

El modelado de comentarios enfrenta la decision clasica entre embedding y referencia. La eleccion depende de la profundidad del arbol y la frecuencia de actualizacion.

Para comentarios de primer nivel (un solo nivel de profundidad), el embedding es la opcion natural:

```json
{
  "_id": ObjectId("..."),
  "articulo_id": ObjectId("..."),
  "titulo": "MongoDB en produccion",
  "contenido": "...",
  "comentarios": [
    {
      "usuario": "user1",
      "texto": "Excelente articulo",
      "fecha": ISODate("2026-07-28T10:00:00Z"),
      "votos": 12
    },
    {
      "usuario": "user2",
      "texto": "Gracias por compartir",
      "fecha": ISODate("2026-07-28T10:05:00Z"),
      "votos": 3
    }
  ],
  "total_comentarios": 2
}
```

Para debates jerarquicos profundos (respuestas a respuestas), el materialized path sobre una coleccion de comentarios ofrece mejor escalabilidad:

```json
{
  "_id": ObjectId("..."),
  "articulo_id": ObjectId("..."),
  "usuario": "user1",
  "texto": "Muy buen punto",
  "path": ",articulo_123,comentario_456,",
  "profundidad": 2,
  "fecha": ISODate("2026-07-28T11:00:00Z")
}
```

Change Streams permiten actualizar en tiempo real los hilos de comentarios en los clientes conectados. Cada vez que se inserta un nuevo comentario, el change stream notifica a los suscriptores, que actualizan la interfaz sin polling:

```javascript
const changeStream = db.comentarios.watch([
  { $match: { "fullDocument.articulo_id": articuloId } }
]);
changeStream.on("change", (change) => {
  // Enviar via WebSocket al cliente
  broadcast(change.fullDocument);
});
```

## Chat y Mensajeria

Los sistemas de mensajeria requieren escrituras de baja latencia, actualizaciones en tiempo real y limpieza automatica de mensajes antiguos. MongoDB encaja naturalmente en este dominio.

El modelo recomendado almacena los mensajes embebidos dentro del documento de la conversacion, con un tope de mensajes por documento (bucket pattern):

```json
{
  "_id": ObjectId("..."),
  "participantes": [ObjectId("user1"), ObjectId("user2")],
  "ultimo_mensaje": {
    "texto": "Te veo manana",
    "usuario_id": ObjectId("user2"),
    "fecha": ISODate("2026-07-28T12:00:00Z")
  },
  "mensajes": [
    {
      "usuario_id": ObjectId("user1"),
      "texto": "Hola, quedamos a las 5?",
      "fecha": ISODate("2026-07-28T11:55:00Z"),
      "tipo": "texto"
    },
    {
      "usuario_id": ObjectId("user2"),
      "texto": "Te veo manana",
      "fecha": ISODate("2026-07-28T12:00:00Z"),
      "tipo": "texto"
    }
  ],
  "fecha_actualizacion": ISODate("2026-07-28T12:00:00Z")
}
```

Cuando el array de mensajes alcanza un tamano limite (por ejemplo, 100 mensajes), se crea un nuevo bucket. La consulta de mensajes se realiza con proyeccion y ordenacion:

```javascript
db.conversaciones.find(
  { _id: conversacionId },
  { mensajes: { $slice: -50 } }
)
```

El indice TTL (Time-To-Live) sobre `fecha_actualizacion` elimina automaticamente las conversaciones inactivas:

```javascript
db.conversaciones.createIndex(
  { fecha_actualizacion: 1 },
  { expireAfterSeconds: 2592000 } // 30 dias
)
```

Change Streams proporcionan las actualizaciones en tiempo real necesarias para la experiencia de chat. Cada cliente se suscribe a los cambios de sus conversaciones y recibe los nuevos mensajes instantaneamente.

## IoT y Sensores

El Internet de las Cosas genera volumes masivos de datos temporales que deben ingerirse, almacenarse y consultarse eficientemente. MongoDB aborda este escenario con el patron Bucket, sharding por rango temporal y TTL automatico.

El esquema de un bucket de sensores combina el patron Bucket con indices compuestos:

```json
{
  "sensor_id": "temp-sensor-01",
  "bucket_inicio": ISODate("2026-07-28T10:00:00Z"),
  "bucket_fin": ISODate("2026-07-28T11:00:00Z"),
  "lecturas": [
    { "t": 0, "v": 23.5 },
    { "t": 60, "v": 23.7 }
  ],
  "estadisticas": {
    "min": 23.1,
    "max": 24.2,
    "promedio": 23.6,
    "total": 60
  }
}
```

Sharding es obligatorio cuando el volumen supera la capacidad de un solo nodo. La clave de sharding mas comun es `{ sensor_id: 1, bucket_inicio: 1 }`, que distribuye los datos por sensor y rango temporal:

```javascript
sh.shardCollection("iot.lecturas", { sensor_id: 1, bucket_inicio: 1 })
```

El indice TTL sobre `bucket_fin` elimina automaticamente los buckets con mas de N dias, implementando una politica de retencion de datos sin intervencion manual:

```javascript
db.lecturas.createIndex(
  { bucket_fin: 1 },
  { expireAfterSeconds: 7776000 } // 90 dias
)
```

Las consultas analiticas sobre ventanas temporales se resuelven con el aggregation pipeline, utilizando `$unwind` sobre el array de lecturas y `$group` para calcular agregaciones.

## Gaming

Las aplicaciones de gaming requieren lecturas de baja latencia para leaderboards en tiempo real, inventarios de jugadores con esquemas flexibles y sesiones que deben persistir sin afectar la experiencia de juego.

Los leaderboards se implementan con un indice descendente sobre la puntuacion:

```javascript
db.jugadores.createIndex({ puntuacion: -1 })
```

La consulta del top 100 es una simple busqueda ordenada:

```javascript
db.jugadores.find(
  {},
  { nombre: 1, puntuacion: 1, nivel: 1 }
).sort({ puntuacion: -1 }).limit(100)
```

Para el ranking posicional de un jugador especifico, se usa un conteo de jugadores con puntuacion superior:

```javascript
db.jugadores.countDocuments({ puntuacion: { $gt: jugador.puntuacion } }) + 1
```

El inventario de cada jugador se modela con documentos embebidos que representan objetos, armas o habilidades. El esquema flexible permite que cada jugador tenga combinaciones unicas de items sin esquemas predefinidos:

```json
{
  "_id": ObjectId("..."),
  "jugador_id": ObjectId("..."),
  "oro": 12500,
  "inventario": [
    { "item_id": "espada-hierro", "cantidad": 1, "durabilidad": 85 },
    { "item_id": "pocion-vida", "cantidad": 12 },
    { "item_id": "llave-dorada", "cantidad": 1 }
  ],
  "misiones_activas": ["salvar-reino", "derrotar-dragon"]
}
```

Las actualizaciones atomicas (`$inc`, `$push`, `$pull`) permiten modificar inventarios en tiempo real sin riesgos de concurrencia.

## Analytics y Business Intelligence

MongoDB no solo es una base de datos operacional, sino que tambien puede actuar como fuente analitica gracias al aggregation pipeline y a la integracion con Atlas y herramientas de BI.

El aggregation pipeline permite transformar datos operacionales en metricas de negocio sin necesidad de ETL externo. Un informe de ventas diarias por categoria:

```javascript
db.pedidos.aggregate([
  { $match: { fecha: { $gte: inicio, $lt: fin } } },
  { $unwind: "$articulos" },
  {
    $group: {
      _id: { categoria: "$articulos.categoria", fecha: { $dateToString: { format: "%Y-%m-%d", date: "$fecha" } } },
      total_ventas: { $sum: { $multiply: ["$articulos.precio", "$articulos.cantidad"] } },
      unidades: { $sum: "$articulos.cantidad" },
      pedidos: { $addToSet: "$_id" }
    }
  },
  { $sort: { "_id.fecha": 1, total_ventas: -1 } },
  { $out: "resumen_ventas_diarias" }
])
```

Atlas Charts proporciona visualizaciones directamente desde los datos sin moverlos, con dashboards que se actualizan automaticamente. Para entornos enterprise, el conector MongoDB BI (que traduce SQL al aggregation pipeline) permite que herramientas como Tableau o PowerBI consulten MongoDB como si fuera una base de datos relacional, exponiendo vistas virtuales sobre el modelo documental.

La combinacion de aggregation pipeline, Atlas Charts y conectores BI convierte a MongoDB en una plataforma unica que elimina la separacion tradicional entre base de datos operacional y almacen analitico, reduciendo la latencia entre la generacion del dato y su disponibilidad para la toma de decisiones.

