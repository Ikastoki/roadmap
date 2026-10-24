# Drivers oficiales de MongoDB

MongoDB expone su protocolo de red (OP_MSG sobre TCP/TLS) a traves de un conjunto de drivers oficiales mantenidos por MongoDB Inc. que implementan el wire protocol, el handshake de autenticacion, la serializacion BSON y las operaciones CRUD, de agregacion y de administracion. Cada driver abstrae las particularidades del lenguaje para ofrecer una experiencia idiomatica: promesas en JavaScript, LINQ en C#, generadores en Python, interfaces fluyentes en Java. La eleccion del driver correcto determina no solo la productividad del equipo, sino el rendimiento, la gestion de conexiones y la capacidad de aprovechar caracteristicas avanzadas como change streams, transactions o gridfs.

## Driver para Java (mongodb-driver-sync)

El driver sincrono de Java es el mas maduro del ecosistema MongoDB. Proporciona una API basada en bloques de construccion (`MongoClient`, `MongoDatabase`, `MongoCollection`) que se alinea con el tipado fuerte de Java. Desde la version 4.x, el driver incorpora el cliente reactivo como modulo independiente, pero el sincrono sigue siendo la opcion mayoritaria en aplicaciones empresariales.

```java
MongoClient client = MongoClients.create("mongodb://localhost:27017");
MongoDatabase db = client.getDatabase("tienda");
MongoCollection<Document> coll = db.getCollection("productos");

// Insertar
Document doc = new Document("nombre", "Laptop").append("precio", 1200);
coll.insertOne(doc);

// Consultar con filtro
Document filtro = new Document("precio", new Document("$gt", 500));
FindIterable<Document> resultados = coll.find(filtro);
for (Document d : resultados) {
    System.out.println(d.toJson());
}
```

El driver sincrono utiliza un pool de conexiones interno que bloquea el hilo durante la operacion de red, lo que lo hace adecuado para aplicaciones que ya usan un pool de hilos propio (como los servidores Jakarta EE/Spring con Tomcat o Jetty). Para escenarios reactivos, MongoDB ofrece `mongodb-driver-reactivestreams`, que implementa el estandar Reactive Streams.

## Driver para Node.js (mongodb)

El driver nativo de Node.js expone una API asincrona basada en callbacks desde sus origenes y evoluciono hacia promesas y `async/await`. Es la capa sobre la que se construye Mongoose y otros ODM. El uso tipico sigue un patron de conexion unica (singleton) que se reutiliza durante todo el ciclo de vida de la aplicacion.

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
await client.connect();

const db = client.db('tienda');
const coll = db.collection('productos');

// Insertar
await coll.insertOne({ nombre: 'Laptop', precio: 1200 });

// Consultar
const resultados = await coll.find({ precio: { $gt: 500 } }).toArray();

// Cerrar al terminar
await client.close();
```

El driver de Node.js maneja automaticamente la reconexion, la seleccion del servidor en replica sets y el balanceo de lecturas segun `readPreference`. El pool de conexiones por defecto es de 100 conexiones, ajustable mediante `maxPoolSize`.

## Driver para Python (pymongo)

PyMongo es el driver oficial sincrono para Python. Su API refleja la nomenclatura del shell de MongoDB y se integra de forma natural con el tipado dinamico del lenguaje. El patron de acceso es similar al de Java y Node.js, pero adaptado a la sintaxis de diccionarios y listas de Python.

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017')
db = client.tienda
coll = db.productos

# Insertar
coll.insert_one({'nombre': 'Laptop', 'precio': 1200})

# Consultar
for doc in coll.find({'precio': {'$gt': 500}}):
    print(doc)
```

PyMongo soporta todas las caracteristicas del wire protocol, incluyendo aggregation pipeline, change streams, transactions y gridfs para almacenamiento de archivos grandes. Para entornos de alta concurrencia donde el GIL de Python puede ser un cuello de botella, se utiliza Motor, el driver asincrono.

## Motor (async con alta concurrencia)

Motor es el driver asincrono oficial para Python, construido sobre PyMongo y la libreria `asyncio`. Proporciona una API identica a la de PyMongo pero con metodos que deben ser `await`eados, permitiendo que un solo hilo gestione miles de conexiones simultaneas sin bloqueo.

```python
from motor.motor_asyncio import AsyncIOMotorClient

client = AsyncIOMotorClient('mongodb://localhost:27017')
db = client.tienda
coll = db.productos

async def ejemplo():
    await coll.insert_one({'nombre': 'Laptop', 'precio': 1200})
    async for doc in coll.find({'precio': {'$gt': 500}}):
        print(doc)
```

Motor es la opcion preferida para aplicaciones web asyncronas basadas en FastAPI, Sanic o AioHTTP, donde cada request no debe bloquear el event loop. Su arquitectura internamente delega las operaciones de red en PyMongo pero utilizando callbacks sobre el event loop de asyncio.

## Driver para C# (mongodb-driver con LINQ)

El driver de C# sigue la convencion .NET de interfaces (`IMongoClient`, `IMongoDatabase`, `IMongoCollection`) y metodos asincronos con el patron `Task<T>`. Su caracteristica mas diferenciadora es el soporte de LINQ: las consultas se expresan como expresiones lambda y el driver las traduce automaticamente al pipeline de agregacion de MongoDB.

```csharp
var client = new MongoClient("mongodb://localhost:27017");
var db = client.GetDatabase("tienda");
var coll = db.GetCollection<BsonDocument>("productos");

// Con LINQ
var resultados = coll.AsQueryable()
    .Where(p => p.GetValue("precio") > 500)
    .ToList();

// Con API fluida
var filtro = Builders<BsonDocument>.Filter.Gt("precio", 500);
var docs = coll.Find(filtro).ToList();
```

La integracion con LINQ permite que desarrolladores acostumbrados a Entity Framework o NHibernate escriban consultas tipadas con verificacion en compilacion, manteniendo la potencia del aggregation framework de MongoDB.

## Driver para Go (mongo-go-driver)

El driver oficial de Go sigue las convenciones del lenguaje: metodos que devuelven `(Result, Error)`, manejo explicito de contextos con `context.Context` y estructuras Go para mapear documentos BSON. La API es fluida y favorece la composicion de opciones mediante el patron functional options.

```go
client, err := mongo.Connect(context.TODO(), options.Client().ApplyURI("mongodb://localhost:27017"))
if err != nil {
    log.Fatal(err)
}
defer client.Disconnect(context.TODO())

coll := client.Database("tienda").Collection("productos")

// Insertar
doc := bson.D{{"nombre", "Laptop"}, {"precio", 1200}}
_, err = coll.InsertOne(context.TODO(), doc)

// Consultar
cursor, err := coll.Find(context.TODO(), bson.D{{"precio", bson.D{{"$gt", 500}}}})
var resultados []bson.M
cursor.All(context.TODO(), &resultados)
```

Go tipifica BSON con `bson.D` (ordenado), `bson.M` (mapa), `bson.A` (array) y `bson.E` (elemento). Esta distincion es crucial porque los documentos BSON preservan el orden de los campos, algo que los mapas de Go no garantizan.

## Driver para Ruby (mongo-ruby-driver)

El driver de Ruby expone una API que sigue la filosofia del lenguaje: bloques, metodos dinamicos y una sintaxis legible. Soporta tanto operaciones sincronas como asincronas y se integra con frameworks web como Ruby on Rails a traves del ODM Mongoid.

```ruby
client = Mongo::Client.new('mongodb://localhost:27017/tienda')
coll = client[:productos]

# Insertar
coll.insert_one({ nombre: 'Laptop', precio: 1200 })

# Consultar
coll.find(precio: { '$gt' => 500 }).each do |doc|
  puts doc
end
```

El driver maneja la reconexion automatica y soporta todas las funcionalidades del wire protocol incluyendo sessions causales, change streams y transacciones.

## Driver para PHP (mongodb-php)

El driver PHP de MongoDB esta compuesto por una extension en C (`mongodb`) que implementa el wire protocol, y una capa de libreria PHP (`mongodb-php-library`) que proporciona la API de alto nivel. Esta separacion permite que la extension C sea eficiente mientras que la libreria PHP ofrece una interfaz comoda y documentada.

```php
$client = new MongoDB\Client("mongodb://localhost:27017");
$coll = $client->tienda->productos;

// Insertar
$coll->insertOne(['nombre' => 'Laptop', 'precio' => 1200]);

// Consultar
$resultados = $coll->find(['precio' => ['$gt' => 500]]);
foreach ($resultados as $doc) {
    echo json_encode($doc) . "\n";
}
```

La extension C expone clases como `MongoDB\Driver\Manager`, `MongoDB\Driver\Command` y `MongoDB\Driver\Query`, mientras que la libreria de alto nivel las envuelve en la interfaz familiar de `MongoDB\Collection`, `MongoDB\Database` y `MongoDB\Client`.

---

> El patron de conexion en todos los drivers sigue la misma recomendacion: crear una unica instancia de `MongoClient` al iniciar la aplicacion y reutilizarla globalmente. El pool interno gestiona las conexiones al replica set o al cluster shardeado, detecta fallos y reconfigura la topologia automaticamente.

