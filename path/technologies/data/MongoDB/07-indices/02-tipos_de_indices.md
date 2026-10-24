# Tipos de indices en MongoDB

MongoDB ofrece una decena de tipos de indice, cada uno disenado para un patron de consulta especifico. La eleccion del tipo correcto determina no solo la velocidad de la consulta, sino tambien la viabilidad de ciertos operadores (full-text, geospatial) que simplemente no funcionan sin el indice apropiado.

## Single field index

El tipo mas basico: indexa un unico campo de la coleccion. Soporta cualquier tipo de dato (numeros, strings, fechas, ObjectId) y cualquier operador de comparacion (`$eq`, `$gt`, `$lt`, `$in`, `$regex` cuando es prefix).

```javascript
db.usuarios.createIndex({ email: 1 })   // ascendente
db.usuarios.createIndex({ createdAt: -1 }) // descendente
```

El indice sobre `_id` se crea automaticamente al crear la coleccion y no puede eliminarse. Es un indice unico por defecto.

## Compound index

Indice que abarca multiples campos. Es el tipo de indice mas potente y el que requiere mas cuidado en el diseno, ya que el orden de los campos determina que consultas puede servir.

```javascript
db.ventas.createIndex({ fecha: -1, total: 1, cliente: 1 })
```

Los indices compuestos se tratan en detalle en el archivo `04-indices_compuestos.md`.

## Multikey index

Cuando se indexa un campo cuyo valor es un array, MongoDB crea automaticamente un indice multikey: genera una entrada en el indice por cada elemento del array. Esto permite buscar documentos donde el array contenga un valor especifico.

```javascript
db.productos.createIndex({ etiquetas: 1 })
// etiquetas: ["electronica", "oferta", "nuevo"]
// Crea entradas para "electronica", "oferta" y "nuevo"
```

Una limitacion importante es que un indice compuesto solo puede tener un campo multikey. No es posible indexar dos arrays distintos en el mismo indice compuesto porque el producto cartesiano creceria descontroladamente.

```javascript
// Error: No se puede crear un indice compuesto con dos arrays
db.coleccion.createIndex({ tags: 1, categorias: 1 })
// "Cannot create index with two multikey fields"
```

## Text index (full-text)

El indice de texto permite busquedas por palabras completas con soporte para stemming (reduccion a raiz lexica), stop words (palabras vacias) y ponderacion por campo. No utiliza B-Tree sino una estructura de posting lists similar a la de Lucene.

```javascript
db.articulos.createIndex(
  { titulo: "text", cuerpo: "text" },
  { weights: { titulo: 10, cuerpo: 1 } }
)
```

La consulta se realiza con el operador `$text`:

```javascript
db.articulos.find({
  $text: { $search: "mongodb indices", $language: "es" }
})
```

Opciones destacadas de `$text`:

- `$search`: terminos a buscar (separados por espacio, OR por defecto).
- `$language`: idioma para stemming y stop words.
- `$caseSensitive`: distincion mayusculas/minusculas (default false).
- `$diacriticSensitive`: distincion de acentos (default false).

Para busquedas con frase exacta se usan comillas dobles:

```javascript
db.articulos.find({ $text: { $search: "\"alta disponibilidad\"" } })
```

> Solo puede existir un indice de texto por coleccion, aunque puede abarcar multiples campos con ponderaciones distintas.

## Geospatial index

MongoDB soporta consultas geograficas mediante dos tipos de indice.

### 2dsphere

Disenado para datos en la superficie de la Tierra. Almacena puntos, lineas y poligonos en formato GeoJSON y soporta consultas de proximidad, interseccion y contencion.

```javascript
db.lugares.createIndex({ ubicacion: "2dsphere" })

// Documento de ejemplo
{
  nombre: "Torre Eiffel",
  ubicacion: {
    type: "Point",
    coordinates: [2.2945, 48.8584]
  }
}
```

Operadores disponibles: `$near`, `$nearSphere`, `$geoWithin`, `$geoIntersects`.

### 2d

Disenado para proyecciones en un plano cartesiano (coordenadas x, y). Adecuado para aplicaciones en espacios reducidos donde la curvatura terrestre es irrelevante.

```javascript
db.tiendas.createIndex({ posicion: "2d" })
```

> `2dsphere` es la opcion recomendada para datos geograficos reales. `2d` solo debe usarse cuando se trabaja en coordenadas planas (ej. mapas de interiores, juegos).

## Hashed index

Indexa el hash del valor del campo en lugar del valor original. Distribuye los valores uniformemente incluso cuando el campo original tiene muy poca cardinalidad o valores secuenciales.

```javascript
db.eventos.createIndex({ usuarioId: "hashed" })
```

Su uso principal es el sharding con clave hash, donde garantiza una distribucion uniforme de los documentos entre los shards. No soporta consultas por rango (`$gt`, `$lt`) ni ordenacion.

## TTL index (expireAfterSeconds)

Un indice TTL (time-to-live) elimina automaticamente los documentos cuando el valor del campo fecha supera el umbral definido en segundos. MongoDB ejecuta un thread en segundo plano (cada 60 segundos) que comprueba y elimina los documentos expirados.

```javascript
db.sesiones.createIndex(
  { ultimoAcceso: 1 },
  { expireAfterSeconds: 3600 }
)
```

El campo debe ser de tipo Date o array de Dates. Las operaciones de eliminacion pueden afectar al rendimiento si hay muchos documentos expirando simultaneamente.

Casos de uso tipicos:

- Sesiones de usuario temporales.
- Tokens de restablecimiento de contrasena.
- Cache de datos temporales.
- Logs y eventos con retencion limitada.

## Wildcard index ($**)

Indexa todos los campos de los documentos (o un subconjunto definido por patron) sin necesidad de conocer la estructura de antemano. Util para colecciones con esquema totalmente dinamico donde no se sabe que campos consultaran los usuarios.

```javascript
db.configuracion.createIndex({ "$**": 1 })
```

Tambien permite restringir por patron o campo:

```javascript
db.productos.createIndex(
  { "$**": 1 },
  { wildcardProjection: { "atributos.*": 1 } }
)
```

> Los wildcard index pueden consumir mucho espacio si los documentos tienen muchos campos. Hay que usarlos con criterio y restringir el alcance con `wildcardProjection` siempre que sea posible.

## Partial index

Indice que solo incluye los documentos que cumplen una condicion (`partialFilterExpression`). Reduce el tamano del indice y el impacto en escritura porque solo se indexan los documentos relevantes.

```javascript
db.pedidos.createIndex(
  { estado: 1, fecha: 1 },
  { partialFilterExpression: { estado: { $ne: "cancelado" } } }
)
```

La condicion del `partialFilterExpression` debe poder evaluarse en tiempo de insercion. Los documentos que no cumplen la condicion simplemente no aparecen en el indice, por lo que las consultas que buscan documentos fuera del filtro parcial hiran a COLLSCAN.

> Para que una consulta utilice un partial index, su filtro debe incluir la condicion del `partialFilterExpression`.

## Sparse index

Similar al partial index pero con una semantica especifica: solo incluye documentos que tienen el campo indexado (es decir, excluye documentos donde el campo es `null` o no existe).

```javascript
db.clientes.createIndex(
  { codigoDescuento: 1 },
  { sparse: true }
)
```

Un indice normal incluye entradas para documentos que carecen del campo (con valor `null`). El sparse index omite esas entradas, reduciendo el tamano del indice cuando el campo es opcional y solo aparece en un subconjunto de documentos.

> La diferencia entre sparse y partial es sutil: sparse excluye solo por ausencia del campo; partial permite cualquier condicion arbitraria.

## Unique index

Garantiza que no existen dos documentos con el mismo valor en el campo (o combinacion de campos) indexado. Si se intenta insertar un duplicado, MongoDB rechaza la operacion con el error `E11000 duplicate key error`.

```javascript
db.usuarios.createIndex(
  { email: 1 },
  { unique: true }
)
```

El indice sobre `_id` es unico por defecto. Los indices unicos pueden ser compuestos:

```javascript
db.inscripciones.createIndex(
  { usuario: 1, curso: 1 },
  { unique: true }
)
// Un usuario no puede inscribirse dos veces al mismo curso
```

> En presencia de `null`, el indice unico considera que `null` es un valor como cualquier otro: solo un documento puede tener `null` en ese campo. Para permitir multiples documentos sin valor hay que combinar `unique` con `sparse`.

