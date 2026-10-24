# Tipos de bases de datos NoSQL

NoSQL no es un modelo unico sino una familia de modelos con diferentes paradigmas de almacenamiento y acceso. Cada tipo responde a un patron de datos distinto y ofrece ventajas especificas segun el caso de uso. Conocer las diferencias entre ellos es esencial para elegir la base de datos adecuada en cada contexto.

## Bases de datos documentales

Almacenan datos en forma de documentos, tipicamente JSON, BSON o XML. Cada documento es una unidad autonoma que contiene todos los datos relacionados, sin necesidad de joins. La estructura interna puede variar entre documentos de la misma coleccion, lo que proporciona una flexibilidad de esquema total.

**Caracteristicas principales:**
- Los datos se organizan en colecciones (equivalentes a tablas) y documentos (equivalentes a filas).
- Cada documento tiene un identificador unico (`_id`).
- Soportan indices secundarios, agregaciones y consultas por campos anidados.
- La denormalizacion es habitual: los datos relacionados se incrustan (embedding) en lugar de referenciarse.

**Casos de uso:** catalogos de productos, sistemas de gestion de contenidos, aplicaciones con esquemas evolutivos, analitica en tiempo real.

**Sistemas destacados:**
- **MongoDB**: el mas popular. Usa BSON (JSON binario), soporta indices secundarios, sharding nativo, replicacion y aggregation framework.
- **CouchDB**: usa JSON puro, ofrece replicacion multimaster y sincronizacion offline. Disenado para aplicaciones web y moviles.
- **Firebase Firestore**: base de datos documental en la nube con sincronizacion en tiempo real.

```javascript
// Ejemplo de documento MongoDB
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "titulo": "NoSQL Distilled",
  "autor": "Pramod Sadalage",
  "anio": 2012,
  "etiquetas": ["nosql", "bases de datos", "arquitectura"],
  "precio": 45.99,
  "disponible": true
}
```

> Los documentos permiten representar datos anidados de forma natural. Una orden de compra con sus lineas se almacena en un unico documento, eliminando la necesidad de tablas separadas con claves foraneas.

## Bases de datos clave-valor

Son el modelo NoSQL mas simple. Almacenan pares clave-valor donde la clave es un identificador unico y el valor es un blob opaco (el motor no interpreta su contenido). El acceso se realiza exclusivamente por clave, lo que proporciona una latencia extremadamente baja.

**Caracteristicas principales:**
- Operaciones basicas: get, set, delete.
- Escalabilidad horizontal casi lineal: solo hay que distribuir las claves entre nodos.
- No soportan consultas por valor, indices secundarios ni joins.
- El valor puede ser cualquier cosa: un string, un JSON, un binario, un contador.

**Casos de uso:** sesiones de usuario, cachés, colas de mensajes, contadores, almacenamiento de configuracion.

**Sistemas destacados:**
- **Redis**: almacena en memoria con persistencia opcional. Soporta estructuras de datos (listas, sets, hashes, sorted sets), pub/sub, TTL y transacciones.
- **DynamoDB**: servicio gestionado de AWS. Escalabilidad automatica, consistencia configurable, integracion con el ecosistema AWS.
- **Riak**: disenado para alta disponibilidad con replicacion multimaster.

```bash
# Ejemplo Redis
SET usuario:1001 '{"nombre": "Ana", "email": "ana@example.com"}'
GET usuario:1001
EXPIRE usuario:1001 3600
```

## Bases de datos columnares

Tambien conocidas como column-family stores. Almacenan los datos por columnas en lugar de por filas, inspiradas en BigTable de Google. Cada fila puede tener un conjunto diferente de columnas, y las columnas se agrupan en familias.

**Caracteristicas principales:**
- Optimizadas para lecturas/escrituras de grandes volumenes de datos con patrones predecibles.
- Alta compresion porque datos del mismo tipo se almacenan juntos.
- Escalabilidad horizontal masiva: disenadas para cientos de nodos.
- Modelo esparcido: las filas pueden tener millones de columnas, la mayoria vacias, sin coste de almacenamiento.

**Casos de uso:** analitica de series temporales, IoT, almacenamiento de eventos, datos de sensores, sistemas de recomendacion.

**Sistemas destacados:**
- **Cassandra**: disenada por Facebook. Ofrece alta disponibilidad sin punto unico de fallo, consistencia configurable (tunable consistency) y lenguaje de consulta similar a SQL (CQL).
- **HBase**: sobre Hadoop/HDFS. Ofrece consistencia fuerte y esta integrada con el ecosistema Hadoop (MapReduce, Spark).
- **ScyllaDB**: compatible con Cassandra pero escrita en C++ para menor latencia.

```sql
-- Ejemplo CQL (Cassandra)
CREATE TABLE usuarios (
  user_id UUID PRIMARY KEY,
  nombre text,
  email text,
  created_at timestamp
);
```

> A diferencia de las bases relacionales, en las columnares cada fila es independiente: no hay un esquema fijo de columnas obligatorias. Dos filas de la misma tabla pueden tener columnas completamente diferentes.

## Bases de datos de grafos

Almacenan entidades (nodos) y relaciones (aristas) como elementos de primera clase. Cada nodo y arista puede tener propiedades (atributos clave-valor). Las relaciones son navegables en ambos sentidos con rendimiento constante, independientemente del tamaño del grafo.

**Caracteristicas principales:**
- Modelo natural para datos altamente interconectados.
- Consultas de recorrido (traversal) eficientes: encontrar el camino mas corto, amigos en comun, deteccion de ciclos.
- Lenguajes de consulta especificos como Cypher (Neo4j) o Gremlin (Apache TinkerPop).
- Escalan horizontalmente con particionamiento, aunque el rendimiento optimo se obtiene en grafos que caben en un unico servidor.

**Casos de uso:** redes sociales, motores de recomendacion, deteccion de fraude, gestion de identidades, analisis de dependencias.

**Sistemas destacados:**
- **Neo4j**: el mas popular. Lenguaje Cypher, soporta ACID, indices, y propiedades en nodos y aristas.
- **ArangoDB**: base de datos multimodelo (documentos, grafos, clave-valor).
- **JanusGraph**: grafos distribuidos sobre Cassandra, HBase o Bigtable.

```cypher
// Ejemplo Cypher (Neo4j)
CREATE (a:Persona {nombre: "Ana"})
CREATE (b:Persona {nombre: "Luis"})
CREATE (a)-[:AMIGO_DE]->(b)
MATCH (a:Persona)-[:AMIGO_DE]->(amigos)
RETURN a.nombre, amigos.nombre
```

## Bases de datos time-series

Optimizadas para almacenar y consultar datos indexados temporalmente: metricas de servidores, lecturas de sensores, cotizaciones bursatiles, logs de aplicaciones. Estan disenadas para la ingestion continua de grandes volumenes de datos con marcas temporales.

**Caracteristicas principales:**
- Compresion agresiva de datos temporales (los valores repetidos o predecibles se comprimen).
- Retencion y borrado automatico por antiguedad (TTL por politica de datos).
- Consultas temporales eficientes: ventanas de tiempo, downsampling, agregaciones por intervalo.
- Escritura secuencial optimizada: los datos se anaden siempre al final (append-only).

**Casos de uso:** monitorizacion de infraestructura (CPU, RAM, red), IoT, finanzas cuantitativas, analitica de aplicaciones.

**Sistemas destacados:**
- **InfluxDB**: lenguaje de consulta Flux/SQL, retention policies, continuous queries.
- **TimescaleDB**: extension de PostgreSQL que combina SQL con optimizaciones time-series.
- **Prometheus**: disenado para monitorizacion de sistemas, con su propio modelo de datos y lenguaje de consulta (PromQL).

```sql
-- Ejemplo InfluxDB (Flux)
from(bucket: "servidores")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu" and r._field == "usage_percent")
  |> aggregateWindow(every: 5m, fn: mean)
```

> Las time-series comparten caracteristicas con las columnares (alta ingestion, compresion) pero se especializan en el eje temporal, ofreciendo funciones nativas de ventana, retencion y downsampling que no existen en otros modelos.

