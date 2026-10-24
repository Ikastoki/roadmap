# Ejemplos Practicos de Patrones en MongoDB

La teoria de los patrones cobra sentido cuando se traduce a esquemas concretos y consultas ejecutables. Este capitulo presenta implementaciones practicas de los patrones mas relevantes, con codigo listo para ejecutar en un entorno MongoDB real.

## Schema Versioning: Migracion Progresiva

**Problema**: Una aplicacion de usuarios almacena inicialmente solo nombre y email. Tras seis meses en produccion con 500.000 documentos, se necesita anadir telefono y direccion sin downtime.

**Esquema inicial (version 1)**:

```json
{
  "_id": ObjectId("..."),
  "nombre": "Ana Garcia",
  "email": "ana@email.com",
  "schema_version": 1
}
```

**Insercion de nuevos usuarios con version 2**:

```javascript
db.usuarios.insertOne({
  nombre: "Carlos Lopez",
  email: "carlos@email.com",
  telefono: "+34 600 111 222",
  direccion: { calle: "Calle Mayor 1", ciudad: "Madrid" },
  schema_version: 2
})
```

**Actualizacion lazy al leer**:

```javascript
function obtenerUsuario(id) {
  const usuario = db.usuarios.findOne({ _id: id });

  if (usuario.schema_version === 1) {
    // Migrar en caliente
    db.usuarios.updateOne(
      { _id: id },
      {
        $set: {
          telefono: null,
          direccion: null,
          schema_version: 2
        }
      }
    );
    usuario.telefono = null;
    usuario.direccion = null;
    usuario.schema_version = 2;
  }

  return usuario;
}
```

**Migracion background con batch**:

```javascript
let batch = db.usuarios.find({ schema_version: 1 }).limit(1000);
while (batch.hasNext()) {
  const ids = batch.map(u => u._id);
  db.usuarios.updateMany(
    { _id: { $in: ids } },
    {
      $set: {
        telefono: null,
        direccion: null,
        schema_version: 2
      }
    }
  );
  batch = db.usuarios.find({ schema_version: 1 }).limit(1000);
}
```

**Indice para monitorizar la migracion**:

```javascript
db.usuarios.createIndex({ schema_version: 1 })
```

La consulta `db.usuarios.countDocuments({ schema_version: 1 })` muestra el progreso de la migracion.

## Bucket: Agregacion de Datos Time-Series

**Problema**: 10.000 sensores enviando una lectura cada 10 segundos. Almacenar cada lectura como documento individual generaria 86,4 millones de documentos al dia.

**Creacion del bucket**:

```javascript
function insertarLectura(sensorId, timestamp, valor) {
  const bucketInicio = new Date(timestamp);
  bucketInicio.setMinutes(0, 0, 0); // Redondear a hora exacta

  const offsetSegundos = Math.floor((timestamp - bucketInicio) / 1000);

  db.lecturas_sensores.updateOne(
    {
      sensor_id: sensorId,
      bucket_inicio: bucketInicio
    },
    {
      $push: { lecturas: { t: offsetSegundos, v: valor } },
      $min: { "estadisticas.min": valor },
      $max: { "estadisticas.max": valor },
      $inc: { "estadisticas.total": 1 },
      $setOnInsert: {
        sensor_id: sensorId,
        bucket_inicio: bucketInicio,
        bucket_fin: new Date(bucketInicio.getTime() + 3600000),
        estadisticas: { min: valor, max: valor, total: 1 }
      }
    },
    { upsert: true }
  );
}
```

**Consulta de media horaria**:

```javascript
db.lecturas_sensores.aggregate([
  { $match: { sensor_id: "sensor-01" } },
  { $unwind: "$lecturas" },
  {
    $group: {
      _id: "$bucket_inicio",
      promedio: { $avg: "$lecturas.v" },
      minimo: { $min: "$lecturas.v" },
      maximo: { $max: "$lecturas.v" },
      muestras: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])
```

**Indices necesarios**:

```javascript
db.lecturas_sensores.createIndex(
  { sensor_id: 1, bucket_inicio: -1 }
)
```

## Computed: Panel de Control en Tiempo Real

**Problema**: Un panel administrativo muestra el total de ventas del dia, el numero de pedidos y el ticket medio. Recalcular estos valores desde cero en cada peticion es inviable cuando hay miles de pedidos por hora.

**Documento de resumen precalculado**:

```json
{
  "_id": "resumen-2026-07-28",
  "fecha": ISODate("2026-07-28T00:00:00Z"),
  "total_ventas": 45230.50,
  "total_pedidos": 312,
  "ticket_medio": 144.97,
  "top_productos": [
    { "producto": "Laptop X", "unidades": 45, "ingresos": 58455 },
    { "producto": "Mouse Y", "unidades": 120, "ingresos": 3600 }
  ],
  "ultima_actualizacion": ISODate("2026-07-28T15:30:00Z")
}
```

**Worker de actualizacion periodica**:

```javascript
function actualizarResumenDiario() {
  const hoy = new Date();
  hoy.setHours(0, 0, 0, 0);

  const resumen = db.pedidos.aggregate([
    { $match: { fecha: { $gte: hoy } } },
    {
      $group: {
        _id: null,
        total_ventas: { $sum: "$total" },
        total_pedidos: { $sum: 1 }
      }
    }
  ]).next();

  db.resumen_ventas.updateOne(
    { _id: `resumen-${hoy.toISOString().slice(0, 10)}` },
    {
      $set: {
        fecha: hoy,
        total_ventas: resumen.total_ventas,
        total_pedidos: resumen.total_pedidos,
        ticket_medio: resumen.total_ventas / resumen.total_pedidos,
        ultima_actualizacion: new Date()
      }
    },
    { upsert: true }
  );
}

// Ejecutar cada 5 minutos
setInterval(actualizarResumenDiario, 300000);
```

## Extended Reference: Catalogo con Categorias

**Problema**: Cada producto tiene una categoria. Mostrar una lista de 100 productos requiere 1 consulta principal + 100 consultas a la coleccion de categorias (o un `$lookup` costoso).

**Esquema con referencia extendida**:

```json
// Coleccion categorias
{
  "_id": ObjectId("cat-electronica"),
  "nombre": "Electronica",
  "slug": "electronica",
  "descripcion": "Productos electronicos"
}

// Coleccion productos (con campos duplicados de categoria)
{
  "_id": ObjectId("..."),
  "nombre": "Laptop X",
  "precio": 1299,
  "categoria_id": ObjectId("cat-electronica"),
  "categoria_nombre": "Electronica",
  "categoria_slug": "electronica"
}
```

**Consulta de listado sin $lookup**:

```javascript
db.productos.find(
  { precio: { $gte: 500, $lte: 2000 } },
  { nombre: 1, precio: 1, categoria_nombre: 1 }
).sort({ precio: 1 }).limit(100)
```

**Actualizacion sincronizada cuando cambia el nombre de la categoria**:

```javascript
db.productos.updateMany(
  { categoria_id: catId },
  { $set: { categoria_nombre: nuevoNombre } }
)
```

## Materialized Path: Arbol de Categorias

**Problema**: Un e-commerce con categorias anidadas hasta 5 niveles necesita mostrar el arbol completo de navegacion y filtrar productos por cualquier categoria incluyendo subcategorias.

**Insercion de categorias con path**:

```javascript
function insertarCategoria(nombre, padreId = null) {
  let path = ",";
  let nivel = 0;

  if (padreId) {
    const padre = db.categorias.findOne({ _id: padreId });
    path = padre.path + padre._id + ",";
    nivel = padre.nivel + 1;
  }

  return db.categorias.insertOne({
    nombre,
    path,
    nivel,
    padre_id: padreId
  });
}

// Construir arbol
insertarCategoria("Electronica");
insertarCategoria("Portatiles", ObjectId("cat-electronica"));
insertarCategoria("Ultrabooks", ObjectId("cat-portatiles"));
```

**Obtener todas las subcategorias**:

```javascript
function subcategorias(categoriaId) {
  const cat = db.categorias.findOne({ _id: categoriaId });
  return db.categorias.find({
    path: { $regex: `^${cat.path}${categoriaId},` }
  }).toArray();
}
```

**Productos de una categoria y sus descendientes**:

```javascript
function productosPorCategoriaYSubcategorias(categoriaId) {
  const cat = db.categorias.findOne({ _id: categoriaId });
  const paths = db.categorias.find({
    path: { $regex: `^${cat.path}${categoriaId},` }
  }).map(c => c._id);

  return db.productos.find({
    categoria_id: { $in: [...paths, categoriaId] }
  });
}
```

**Indice para busqueda por path**:

```javascript
db.categorias.createIndex({ path: 1 })
```

Aunque `path` contiene una expresion regular, el indice ayuda cuando la busqueda es por prefijo exacto (ej: `path: { $regex: /^,electronica,/ }`).

## Attribute: Filtros Dinamicos

**Problema**: Un marketplace con miles de vendedores y millones de productos necesita permitir filtros por atributos especificos de cada categoria sin crear cientos de campos.

**Indice multikey para atributos**:

```javascript
db.productos.createIndex({ "atributos.k": 1, "atributos.v": 1 })
```

**Busqueda de productos con filtros dinamicos**:

```javascript
function buscarProductos(filtros) {
  const consulta = {};

  if (filtros.atributos) {
    consulta.$and = filtros.atributos.map(attr => ({
      atributos: {
        $elemMatch: { k: attr.clave, v: attr.valor }
      }
    }));
  }

  if (filtros.precio_min || filtros.precio_max) {
    consulta.precio = {};
    if (filtros.precio_min) consulta.precio.$gte = filtros.precio_min;
    if (filtros.precio_max) consulta.precio.$lte = filtros.precio_max;
  }

  return db.productos.find(consulta).limit(50).toArray();
}

// Ejemplo: portatiles con 32GB RAM entre 800 y 2000 euros
buscarProductos({
  atributos: [{ clave: "ram", valor: "32GB" }],
  precio_min: 800,
  precio_max: 2000
})
```

## Outlier: Usuarios con Millones de Seguidores

**Problema**: El 99% de los usuarios tiene menos de 1000 seguidores, pero 10 usuarios celebricad tienen mas de 10 millones. Almacenar todos los seguidores en el documento del usuario superaria el limite de 16 MB.

**Esquema dual**:

```javascript
function esOutlier(usuarioId) {
  return db.seguidores.countDocuments({ usuario_id: usuarioId }) > 100000;
}

function obtenerSeguidores(usuarioId) {
  if (esOutlier(usuarioId)) {
    return db.seguidores_outlier.findOne(
      { usuario_id: usuarioId },
      { seguidores: 1 }
    ).seguidores;
  }

  return db.usuarios.findOne(
    { _id: usuarioId },
    { seguidores: 1 }
  ).seguidores;
}
```

**Indice para deteccion rapida de outliers**:

```javascript
db.usuarios.createIndex({ num_seguidores: 1 })
db.usuarios.createIndex(
  { es_outlier: 1 },
  { sparse: true, partialFilterExpression: { es_outlier: true } }
)
```

## Tree con $graphLookup: Comentarios Anidados

**Problema**: Un sistema de comentarios con profundidad variable (respuestas a respuestas) donde los arboles deben poder reordenarse y editarse sin reescribir paths completos.

**Esquema con referencia al padre (adjacency list)**:

```json
{
  "_id": ObjectId("comentario-001"),
  "articulo_id": ObjectId("articulo-123"),
  "usuario": "ana",
  "texto": "Articulo muy completo",
  "padre_id": null,
  "fecha": ISODate("2026-07-28T10:00:00Z")
}
{
  "_id": ObjectId("comentario-002"),
  "articulo_id": ObjectId("articulo-123"),
  "usuario": "pedro",
  "texto": "Totalmente de acuerdo",
  "padre_id": ObjectId("comentario-001"),
  "fecha": ISODate("2026-07-28T10:05:00Z")
}
```

**Consulta con $graphLookup para obtener el arbol completo**:

```javascript
db.comentarios.aggregate([
  { $match: { articulo_id: ObjectId("articulo-123") } },
  {
    $graphLookup: {
      from: "comentarios",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "padre_id",
      maxDepth: 10,
      depthField: "nivel",
      as: "respuestas"
    }
  },
  { $sort: { fecha: -1 } }
])
```

**Indices para este patron**:

```javascript
db.comentarios.createIndex({ articulo_id: 1, fecha: -1 })
db.comentarios.createIndex({ padre_id: 1 })
```

El patron adjacency list con `$graphLookup` es preferible al materialized path cuando los arboles se reestructuran con frecuencia, ya que mover un subarbol solo requiere actualizar el `padre_id` del nodo raiz, no reescribir los paths de todos los descendientes.

