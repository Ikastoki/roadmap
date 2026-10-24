# Insertar documentos

MongoDB almacena datos en documentos BSON dentro de colecciones. La operacion de escritura mas elemental es la insercion, y a diferencia de SQL donde hay que definir un esquema previamente, en MongoDB las colecciones se crean implicitamente en el primer insert. No hay restricciones de columnas, tipos ni claves foraneas: cualquier documento con cualquier estructura puede insertarse en cualquier coleccion.

## insertOne

El metodo `insertOne` inserta un unico documento en la coleccion. Si el documento no incluye el campo `_id`, MongoDB lo genera automaticamente como un ObjectId de 12 bytes.

```javascript
db.usuarios.insertOne({
  nombre: "Ana",
  email: "ana@example.com",
  edad: 28,
  activo: true
})
```

La operacion devuelve un objeto con `acknowledged: true` y el `insertedId` generado:

```javascript
{ acknowledged: true, insertedId: ObjectId("665a1b2c3d4e5f6a7b8c9d0e") }
```

El `_id` puede especificarse manualmente, pero debe ser unico dentro de la coleccion. Si se intenta insertar un documento con un `_id` ya existente, MongoDB lanza un error de clave duplicada (error 11000):

```javascript
db.usuarios.insertOne({ _id: "user-001", nombre: "Luis" })
```

> El `_id` puede ser de cualquier tipo BSON: ObjectId, string, number, UUID o incluso un documento embebido. Lo unico que se exige es que sea unico dentro de la coleccion.

## insertMany

`insertMany` permite insertar multiples documentos en una sola operacion, lo que reduce el numero de viajes de ida y vuelta al servidor respecto a ejecutar varios `insertOne` secuenciales.

```javascript
db.ventas.insertMany([
  { producto: "laptop", precio: 1200, cantidad: 2 },
  { producto: "raton", precio: 25, cantidad: 5 },
  { producto: "teclado", precio: 80, cantidad: 3 }
])
```

El resultado incluye los `insertedIds` de todos los documentos insertados:

```javascript
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId("665a1b2c3d4e5f6a7b8c9d0f"),
    '1': ObjectId("665a1b2c3d4e5f6a7b8c9d10"),
    '2': ObjectId("665a1b2c3d4e5f6a7b8c9d11")
  }
}
```

Por defecto, `insertMany` ejecuta las inserciones en modo **ordered**: si falla la insercion de un documento (por ejemplo, por violacion de clave unica), la operacion se detiene y los documentos posteriores no se insertan. Para cambiar este comportamiento se pasa `{ ordered: false }` como segundo argumento:

```javascript
db.ventas.insertMany([
  { _id: 1, producto: "laptop", precio: 1200 },
  { _id: 1, producto: "raton", precio: 25 },   // duplicate key
  { _id: 3, producto: "teclado", precio: 80 }
], { ordered: false })
```

En modo unordered, MongoDB intenta insertar todos los documentos independientemente de los errores. Los documentos con error se ignoran y el resto se insertan. Esto es util cuando se importan lotes de datos donde algunos registros pueden estar corruptos o duplicados.

## Bulk writes

Las operaciones bulk van mas alla de `insertMany`: permiten mezclar inserts, updates y deletes en una sola llamada. MongoDB expone dos enfoques para operaciones bulk: el metodo `bulkWrite()` y el constructor `initializeUnorderedBulkOp`/`initializeOrderedBulkOp`.

### bulkWrite

`bulkWrite` acepta un array de operaciones, cada una identificada por su tipo:

```javascript
db.productos.bulkWrite([
  { insertOne: { document: { nombre: "monitor", precio: 300 } } },
  { updateOne: {
      filter: { nombre: "laptop" },
      update: { $set: { precio: 1100 } }
  }},
  { deleteOne: {
      filter: { nombre: "raton" }
  }}
])
```

### Operaciones soportadas en bulkWrite

Cada elemento del array debe ser un documento con una unica clave que indique el tipo de operacion:

```javascript
{ insertOne: { document: <doc> } }
{ updateOne: { filter: <query>, update: <update>, upsert: <boolean> } }
{ updateMany: { filter: <query>, update: <update>, upsert: <boolean> } }
{ replaceOne: { filter: <query>, replacement: <doc>, upsert: <boolean> } }
{ deleteOne: { filter: <query> } }
{ deleteMany: { filter: <query> } }
```

### Orden de ejecucion: ordered vs unordered

El parametro `ordered` determina como maneja MongoDB los errores dentro de un bulk:

**Ordered (por defecto: `ordered: true`)**:
- Las operaciones se ejecutan secuencialmente en el orden especificado.
- Si una operacion falla, MongoDB detiene el bulk y no ejecuta las operaciones restantes.
- El documento de error incluye el indice de la operacion que fallo.
- Es util cuando las operaciones dependen entre si o se necesita detenerse ante el primer error.

```javascript
db.ventas.bulkWrite([
  { insertOne: { document: { _id: 1, item: "A" } } },
  { insertOne: { document: { _id: 1, item: "B" } } },  // falla
  { insertOne: { document: { _id: 3, item: "C" } } }   // no se ejecuta
])  // Error: duplicate key
```

**Unordered (`ordered: false`)**:
- MongoDB puede ejecutar las operaciones en paralelo, sin garantia de orden.
- Si una operacion falla, las demas continuan ejecutandose.
- Al finalizar, se reportan todos los errores ocurridos.
- Ofrece mejor rendimiento porque MongoDB puede agrupar y paralelizar operaciones del mismo tipo.

```javascript
db.ventas.bulkWrite([
  { insertOne: { document: { _id: 1, item: "A" } } },
  { insertOne: { document: { _id: 1, item: "B" } } },  // falla
  { insertOne: { document: { _id: 3, item: "C" } } }   // se ejecuta
], { ordered: false })
// Resultado: se insertan A y C, B falla
```

> En modo unordered, las operaciones del mismo tipo (por ejemplo, varias inserciones) se agrupan y se envian al servidor como un lote, lo que reduce la sobrecarga de red. En modo ordered, cada operacion se envia individualmente en orden.

### initializeOrderedBulkOp / initializeUnorderedBulkOp

Antes de `bulkWrite`, la forma de hacer operaciones bulk era mediante el constructor del `Bulk`:

```javascript
const bulk = db.productos.initializeUnorderedBulkOp()
bulk.insert({ nombre: "monitor", precio: 300 })
bulk.find({ nombre: "laptop" }).updateOne({ $set: { precio: 1100 } })
bulk.find({ nombre: "raton" }).deleteOne()
bulk.execute()
```

Este enfoque sigue funcionando, pero `bulkWrite` es la forma recomendada por ser mas concisa y explicita.

### Rendimiento y consideraciones

- Cada operacion bulk puede contener un maximo de 100.000 operaciones (configurable via `maxWriteBatchSize`).
- El tamano total del mensaje no debe superar los 48MB (limite de mensaje BSON).
- Las operaciones bulk son atomicas a nivel de documento, pero no a nivel de batch: si falla una operacion en medio de un bulk ordered, las operaciones anteriores se mantienen.
- Para inserciones masivas donde no se necesita feedback individual, `insertMany` es ligeramente mas eficiente que `bulkWrite` con inserts.

> Aunque `bulkWrite` acepte multiples tipos de operacion, no es una transaccion: si se necesita atomicidad entre documentos distintos, hay que usar transacciones multi-documento.

