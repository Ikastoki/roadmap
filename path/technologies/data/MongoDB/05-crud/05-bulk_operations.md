# Bulk operations

Cuando una aplicacion necesita ejecutar multiples operaciones de escritura, hacerlo individualmente genera una sobrecarga de red considerable: cada operacion implica un viaje de ida y vuelta al servidor, la validacion individual del mensaje BSON y la espera de la confirmacion. Las operaciones bulk agrupan multiples escrituras en un solo mensaje, reduciendo drastricamente la latencia y mejorando el rendimiento general.

## bulkWrite

El metodo `bulkWrite` es la forma moderna de ejecutar operaciones bulk en MongoDB. Acepta un array de operaciones y, opcionalmente, un objeto de opciones.

```javascript
db.ventas.bulkWrite([
  { insertOne: { document: { producto: "laptop", precio: 1200 } } },
  { updateOne: {
      filter: { producto: "monitor" },
      update: { $set: { precio: 350 } }
  }},
  { deleteOne: {
      filter: { producto: "raton" }
  }}
])
```

Cada elemento del array debe ser un objeto con exactamente una clave que identifica el tipo de operacion.

### Estructura de cada operacion

**insertOne**:

```javascript
{ insertOne: { document: { ... } } }
```

**updateOne**:

```javascript
{ updateOne: { filter: { ... }, update: { ... }, upsert: <boolean>, collation: { ... }, arrayFilters: [...] } }
```

**updateMany**:

```javascript
{ updateMany: { filter: { ... }, update: { ... }, upsert: <boolean>, collation: { ... }, arrayFilters: [...] } }
```

**replaceOne**:

```javascript
{ replaceOne: { filter: { ... }, replacement: { ... }, upsert: <boolean>, collation: { ... } } }
```

**deleteOne**:

```javascript
{ deleteOne: { filter: { ... }, collation: { ... } } }
```

**deleteMany**:

```javascript
{ deleteMany: { filter: { ... }, collation: { ... } } }
```

## Ordered vs unordered

El parametro `ordered` determina el comportamiento del bulk ante errores y, en cierta medida, el rendimiento.

### Ordered (por defecto)

```javascript
db.ventas.bulkWrite([
  { insertOne: { document: { _id: 1, item: "A" } } },
  { insertOne: { document: { _id: 2, item: "B" } } },
  { insertOne: { document: { _id: 1, item: "C" } } },  // error: duplicado
  { insertOne: { document: { _id: 4, item: "D" } } }   // no se ejecuta
])
```

Comportamiento:
- Las operaciones se ejecutan secuencialmente en el orden en que se definieron.
- Si una operacion falla, MongoDB detiene inmediatamente la ejecucion del bulk.
- Las operaciones anteriores al error se mantienen (no hay rollback).
- El error incluye el indice de la operacion que fallo y el codigo de error.

A nivel interno, MongoDB agrupa las operaciones en lotes de hasta 100.000 operaciones o 48MB. En modo ordered, los lotes se envian secuencialmente, esperando la confirmacion de cada uno antes de enviar el siguiente.

### Unordered

```javascript
db.ventas.bulkWrite([
  { insertOne: { document: { _id: 1, item: "A" } } },
  { insertOne: { document: { _id: 2, item: "B" } } },
  { insertOne: { document: { _id: 1, item: "C" } } },  // error: duplicado
  { insertOne: { document: { _id: 4, item: "D" } } }   // se ejecuta
], { ordered: false })
```

Comportamiento:
- MongoDB puede reorganizar y paralelizar las operaciones para maximizar el rendimiento.
- Si una operacion falla, las demas continuan ejecutandose.
- Al finalizar, se reportan todos los errores ocurridos, no solo el primero.
- Las operaciones del mismo tipo se agrupan en lotes que se ejecutan en paralelo.

El resultado en modo unordered incluye todas las operaciones que se intentaron, con sus respectivos resultados:

```javascript
{
  acknowledged: true,
  insertedCount: 3,
  insertedIds: { '0': 1, '1': 2, '3': 4 },
  matchedCount: 0,
  modifiedCount: 0,
  deletedCount: 0,
  upsertedCount: 0,
  upsertedIds: {},
  writeErrors: [
    {
      index: 2,
      code: 11000,
      errmsg: "E11000 duplicate key error collection: test.ventas index: _id_ dup key: { _id: 1 }"
    }
  ]
}
```

### Cuando usar cada modo

**Ordered**:
- Las operaciones tienen dependencias entre si (por ejemplo, insertar un documento y luego actualizarlo).
- Se necesita detenerse ante el primer error para evitar propagar datos incorrectos.
- La integridad del lote es mas importante que el rendimiento.

**Unordered**:
- Las operaciones son independientes entre si.
- Se prioriza el rendimiento sobre la detencion ante errores.
- Se importan datos donde algunos registros pueden ser invalidos y se quiere insertar el resto.
- El caso tipico: migraciones de datos, importaciones masivas, sincronizacion de sistemas.

> En modo unordered, MongoDB puede ejecutar lotes en paralelo, lo que ofrece un rendimiento significativamente mayor que ordered para grandes volumenes de datos. La diferencia se acentua cuando hay latencia de red de por medio.

## Rendimiento y limites

Las operaciones bulk estan disenadas para alto rendimiento, pero tienen limites que deben conocerse:

```javascript
const docs = []
for (let i = 0; i < 150000; i++) {
  docs.push({ insertOne: { document: { index: i, valor: Math.random() } } })
}

db.coleccion.bulkWrite(docs, { ordered: false })
```

**Limites**:
- **Maximo de operaciones por bulk**: 100.000 (configurable via `maxWriteBatchSize`).
- **Tamano maximo del mensaje**: 48MB (limite de BSON).
- **Tamano maximo de documento**: 16MB por documento individual.

Si se superan estos limites, MongoDB divide automaticamente el bulk en lotes mas pequenos. Esto es transparente para el cliente, pero puede afectar al rendimiento porque cada lote requiere su propio viaje de ida y vuelta.

> Para conjuntos de datos muy grandes (millones de documentos), es recomendable dividir manualmente el bulk en lotes de 10.000 a 50.000 operaciones y enviarlos secuencialmente. Esto evita que un error en medio del lote obligue a reiniciar desde cero.

## initializeOrderedBulkOp / initializeUnorderedBulkOp

Antes de que `bulkWrite` estuviera disponible, las operaciones bulk se construian con los metodos `initializeOrderedBulkOp` e `initializeUnorderedBulkOp` del objeto `db.collection`:

```javascript
const bulk = db.productos.initializeOrderedBulkOp()
bulk.insert({ nombre: "monitor", precio: 300 })
bulk.find({ nombre: "laptop" }).updateOne({ $set: { precio: 1100 } })
bulk.find({ nombre: "raton" }).deleteOne()
const resultado = bulk.execute()
```

Este enfoque sigue siendo funcional, pero `bulkWrite` es preferible por:
- Sintaxis mas concisa y declarativa.
- Mejor integracion con drivers modernos.
- Facilidad para construir arrays de operaciones programaticamente.

## Bulk operations con drivers

En aplicaciones reales, las operaciones bulk se utilizan desde los drivers de MongoDB. La API es similar en todos los lenguajes:

**Node.js**:

```javascript
const result = await collection.bulkWrite([
  { insertOne: { document: { nombre: "Ana", edad: 28 } } },
  { updateOne: {
      filter: { nombre: "Luis" },
      update: { $set: { activo: false } }
  }}
], { ordered: false })
```

**Python (PyMongo)**:

```python
result = db.ventas.bulk_write([
    InsertOne({ "producto": "laptop", "precio": 1200 }),
    UpdateOne({ "producto": "monitor" }, { "$set": { "precio": 350 } }),
    DeleteOne({ "producto": "raton" })
], ordered=False)
```

**Java**:

```java
BulkWriteResult result = collection.bulkWrite(
    Arrays.asList(
        new InsertOneModel<>(new Document("producto", "laptop")),
        new UpdateOneModel<>(
            new Document("producto", "monitor"),
            new Document("$set", new Document("precio", 350))),
        new DeleteOneModel<>(new Document("producto", "raton"))
    ),
    new BulkWriteOptions().ordered(false)
);
```

## Caso practico: sincronizacion masiva

Un caso tipico de uso de bulk operations es la sincronizacion periodica de datos entre sistemas. Supongamos que se recibe un archivo con el catalogo de productos actualizado y se necesita sincronizarlo con la base de datos:

```javascript
function sincronizarCatalogo(productosActualizados) {
  const operations = productosActualizados.map(p => ({
    replaceOne: {
      filter: { sku: p.sku },
      replacement: p,
      upsert: true
    }
  }))

  // Ejecutar en lotes de 10.000
  while (operations.length > 0) {
    const lote = operations.splice(0, 10000)
    db.productos.bulkWrite(lote, { ordered: false })
  }
}
```

Este patron reemplaza cada documento existente por la version actualizada (usando `sku` como identificador unico) e inserta los productos nuevos con `upsert: true`. El modo unordered permite que el resto del lote se procese aunque algun producto tenga errores de validacion.

