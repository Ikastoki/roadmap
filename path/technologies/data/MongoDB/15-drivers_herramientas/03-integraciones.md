# Integraciones de MongoDB

MongoDB no opera en el vacio: en la mayoria de las arquitecturas modernas convive con un ecosistema de herramientas que extienden sus capacidades. El conector BI permite consultar MongoDB con SQL desde herramientas de inteligencia de negocio. Los ODMs (Object Document Mappers) anaden capas de abstraccion y validacion sobre los drivers nativos. Y las integraciones con Kafka, Elasticsearch, Redis y GraphQL habilitan patrones arquitectonicos como la sincronizacion en tiempo real, la cache distribuida, la busqueda full-text y la exposicion de APIs unificadas.

## MongoDB Connector for BI

El conector BI es un servicio independiente (`mongosqld`) que traduce consultas SQL al aggregation pipeline de MongoDB. Actua como un proxy: los clientes SQL (Tableau, Power BI, Looker) se conectan a el como si fuera una base de datos MySQL o PostgreSQL, y el conector convierte las consultas en pipelines de agregacion que se ejecutan en el cluster MongoDB.

```bash
# Iniciar el conector BI
mongosqld --mongo-uri "mongodb://localhost:27017" --schema-dir /esquemas

# Conectar desde MySQL CLI
mysql -h localhost -P 3307 -u root
```

El conector necesita un esquema que describa la estructura de las colecciones MongoDB en terminos relacionales (tablas, columnas, tipos). Este esquema puede generarse automaticamente con `mongo-drdl`:

```bash
mongo-drdl --host localhost:27017 --db tienda --out esquema.drdl
```

El DRDL (Document Relational Definition Language) define la correspondencia entre documentos y tablas, incluyendo como se desanidan los arrays y objetos embebidos. Esquemas tipicos incluyen colecciones que se mapean a tablas individuales y arrays que se desanidan a tablas separadas con claves foraneas virtuales.

```sql
-- Ejemplo de consulta SQL que el conector traduce a aggregation
SELECT categoria, AVG(precio) as precio_medio
FROM productos
WHERE disponible = true
GROUP BY categoria
HAVING AVG(precio) > 100;
```

> El conector BI no esta disenado para reemplazar la API nativa de MongoDB en aplicaciones. Su proposito es habilitar herramientas de reporting y analitica que solo hablan SQL, no para servir trafico de produccion.

## ODMs (Object Document Mappers)

Los ODMs son el equivalente de los ORMs en el mundo relacional: proporcionan una capa de abstraccion que permite trabajar con documentos como objetos nativos del lenguaje de programacion. MongoDB tiene ODMs oficiales y de la comunidad para los lenguajes principales.

### Mongoose (Node.js)

Mongoose es el ODM mas utilizado en el ecosistema Node.js. Anade esquemas tipados, validacion, middlewares (hooks), metodos estaticos, metodos de instancia, virtuals y poblacion de referencias.

```javascript
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const productoSchema = new Schema({
  nombre:     { type: String, required: true, index: true },
  precio:     { type: Number, required: true, min: 0 },
  categoria:  { type: String, enum: ['electronica', 'ropa', 'hogar'] },
  disponible: { type: Boolean, default: true },
  creadoEn:   { type: Date, default: Date.now }
});

// Middleware pre-save
productoSchema.pre('save', function(next) {
  this.nombre = this.nombre.trim();
  next();
});

// Metodo de instancia
productoSchema.methods.aplicarDescuento = function(porcentaje) {
  this.precio = this.precio * (1 - porcentaje / 100);
  return this.precio;
};

const Producto = mongoose.model('Producto', productoSchema);

// Uso
const laptop = new Producto({ nombre: 'Laptop', precio: 1200, categoria: 'electronica' });
await laptop.save();
```

Mongoose soporta poblacion de referencias (el equivalente a `$lookup` a nivel de ODM) con el metodo `populate()`, y validacion basada en esquemas con tipos, rangos, enum y custom validators.

### Morphia (Java)

Morphia es el ODM oficial para Java. Anota clases POJO con metadatos que definen la correspondencia con colecciones MongoDB. Usa anotaciones como `@Entity`, `@Id`, `@Field` y `@Reference`.

```java
@Entity("productos")
public class Producto {
    @Id
    private ObjectId id;
    private String nombre;
    private double precio;
    @Property("categoria")
    private String categoria;
    @Reference
    private List<Proveedor> proveedores;
}

Datastore datastore = Morphia.createDatastore(client, "tienda");
datastore.save(new Producto("Laptop", 1200, "electronica"));

Query<Producto> query = datastore.find(Producto.class)
    .filter("precio >", 500);
List<Producto> resultados = query.iterator().toList();
```

### Spring Data MongoDB

Spring Data MongoDB integra MongoDB con el ecosistema Spring. Proporciona repositorios automaticos (interfaces que Spring implementa en tiempo de ejecucion), soporte para `MongoTemplate` y `MongoRepository`, y transacciones con `@Transactional`.

```java
@Document(collection = "productos")
public class Producto {
    @Id
    private String id;
    private String nombre;
    private Double precio;
    private String categoria;
}

public interface ProductoRepository extends MongoRepository<Producto, String> {
    List<Producto> findByCategoria(String categoria);
    List<Producto> findByPrecioBetween(Double min, Double max);
}

// Uso
@Autowired
private ProductoRepository repo;

List<Producto> electronicos = repo.findByCategoria("electronica");
```

Spring Data MongoDB tambien soporta `@Aggregation` para pipelines directamente desde el repositorio, `@GeoNear` para consultas geoespaciales y `@TextIndex` para busqueda full-text.

### Prisma + MongoDB

Prisma es un ORM moderno que soporta MongoDB como backend desde la version 4.0. Define el esquema en un archivo declarativo (`schema.prisma`) y genera un cliente TypeScript tipado.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model Producto {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  nombre    String
  precio    Float
  categoria String
  disponible Boolean @default(true)
  createdAt DateTime @default(now())
}
```

Prisma traduce las operaciones declarativas a comandos nativos de MongoDB, proporcionando typesafety completa en TypeScript.

## Integracion con Kafka (Change Streams)

La combinacion de change streams de MongoDB con Apache Kafka permite construir pipelines de datos en tiempo real: cada insercion, actualizacion o eliminacion en MongoDB se convierte en un mensaje en un topic de Kafka, que otros consumidores pueden procesar.

```javascript
// Productor: change stream -> Kafka
const { MongoClient } = require('mongodb');
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const producer = kafka.producer();

async function main() {
  const client = await MongoClient.connect('mongodb://localhost:27017');
  const coll = client.db('tienda').collection('productos');
  const changeStream = coll.watch();

  await producer.connect();

  for await (const change of changeStream) {
    await producer.send({
      topic: 'mongodb.changes',
      messages: [{ key: change.documentKey._id.toString(), value: JSON.stringify(change) }]
    });
  }
}
```

El conector oficial `kafka-connect-mongodb` (MongoDB Kafka Connector) proporciona un sink connector (escribe datos de Kafka a MongoDB) y un source connector (publica cambios de MongoDB en Kafka) sin escribir codigo.

> El source connector de MongoDB para Kafka utiliza change streams internamente pero gestiona automaticamente los resume tokens, la reconexion y la particion de topics por operacion, coleccion o base de datos.

## Integracion con Elasticsearch

La integracion de MongoDB con Elasticsearch permite indexar documentos en un motor de busqueda full-text. Mientras que MongoDB tiene indices de texto funcionales, Elasticsearch ofrece capacidades avanzadas de busqueda (fuzzy matching, autocompletado, sinonimia, busqueda por relevancia, faceted search) que lo hacen superior para aplicaciones centradas en busqueda.

El patron tipico utiliza change streams para escuchar cambios en MongoDB y reindexar en Elasticsearch:

```javascript
const { Client } = require('@elastic/elasticsearch');
const { MongoClient } = require('mongodb');

const es = new Client({ node: 'http://localhost:9200' });

async function indexar() {
  const client = await MongoClient.connect('mongodb://localhost:27017');
  const coll = client.db('tienda').collection('productos');
  const changeStream = coll.watch();

  for await (const change of changeStream) {
    if (change.operationType === 'insert' || change.operationType === 'update') {
      await es.index({
        index: 'productos',
        id: change.documentKey._id.toString(),
        body: change.fullDocument
      });
    } else if (change.operationType === 'delete') {
      await es.delete({
        index: 'productos',
        id: change.documentKey._id.toString()
      });
    }
  }
}
```

Existen conectores gestionados como `mongo-connector` (proyecto opensource) y soluciones SaaS como la integracion nativa de MongoDB Atlas Search, que incorpora Apache Lucene directamente en MongoDB sin necesidad de un cluster Elasticsearch separado.

## Integracion con Redis

Redis se utiliza como capa de cache frente a MongoDB para reducir la latencia de lecturas frecuentes y aliviar la carga del servidor de base de datos. El patron tipico es cache-aside: la aplicacion consulta primero Redis; si no encuentra el dato (cache miss), consulta MongoDB y escribe el resultado en Redis con un TTL.

```javascript
const redis = require('redis');
const { MongoClient } = require('mongodb');

const cache = redis.createClient();
const client = await MongoClient.connect('mongodb://localhost:27017');
const coll = client.db('tienda').collection('productos');

async function getProducto(id) {
  // Intentar cache
  const cached = await cache.get(`producto:${id}`);
  if (cached) return JSON.parse(cached);

  // Cache miss: consultar MongoDB
  const doc = await coll.findOne({ _id: id });
  if (doc) {
    await cache.setEx(`producto:${id}`, 3600, JSON.stringify(doc));
  }
  return doc;
}
```

La cache con Redis es especialmente efectiva cuando:
- Los datos son leidos con mucha frecuencia pero escritos con poca frecuencia (catalogos, configuraciones, perfiles).
- Los datos no requieren consistencia inmediata (la cache puede estar desactualizada unos segundos).
- La latencia de lectura de MongoDB es alta debido a consultas complejas o alta carga.

Para invalidar la cache automaticamente al modificar datos en MongoDB, se combina con change streams: cuando un documento se actualiza, se elimina su entrada en Redis.

## Integracion con GraphQL

GraphQL actua como capa de API sobre MongoDB, permitiendo que los clientes soliciten exactamente los datos que necesitan y nada mas. A diferencia de las APIs REST, donde cada endpoint devuelve una estructura fija, GraphQL delega en el cliente la definicion de la respuesta.

```graphql
type Producto {
  id: ID!
  nombre: String!
  precio: Float!
  categoria: String
  disponible: Boolean
}

type Query {
  productos(categoria: String, precioMin: Float, precioMax: Float): [Producto]
  producto(id: ID!): Producto
}
```

La resolucion de estas queries en el backend se traduce a consultas MongoDB. Herramientas como `graphql-mongodb` o `typegraphql` simplifican este mapeo:

```typescript
@Resolver()
class ProductoResolver {
  @Query(() => [Producto])
  async productos(
    @Arg('categoria', { nullable: true }) categoria?: string,
    @Arg('precioMin', { nullable: true }) precioMin?: number
  ) {
    const filtro: any = {};
    if (categoria) filtro.categoria = categoria;
    if (precioMin) filtro.precio = { $gte: precioMin };
    return await Producto.find(filtro).exec(); // Mongoose
  }
}
```

Hasura es una plataforma que expone automaticamente una API GraphQL a partir de una base de datos MongoDB (o PostgreSQL), generando las queries, mutaciones y subscriptions basandose en el esquema de la base de datos.

