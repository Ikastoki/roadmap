# Consultar documentos con find

La lectura de documentos en MongoDB se realiza principalmente mediante el metodo `find`, que acepta un filtro de consulta y una proyeccion opcional. `find` devuelve un cursor que permite iterar sobre los resultados, no los documentos directamente, lo que significa que los datos no se transfieren al cliente hasta que se recorren.

## Sintaxis basica

```javascript
db.usuarios.find()                    // todos los documentos
db.usuarios.find({})                  // equivalente
db.usuarios.find({ edad: 30 })       // igualdad exacta
```

El cursor devuelto puede transformarse en array con `toArray()` o visualizarse con `pretty()`:

```javascript
db.usuarios.find().pretty()
db.usuarios.find().toArray()
```

> `findOne` es un atajo que combina `find` con `limit(1)` y devuelve el primer documento directamente, sin cursor.

## Operadores de comparacion

MongoDB proporciona operadores que permiten filtrar documentos con condiciones mas alla de la igualdad exacta. Se especifican como documentos anidados dentro del campo a evaluar.

```javascript
// Mayor que
db.productos.find({ precio: { $gt: 100 } })

// Menor que
db.productos.find({ precio: { $lt: 50 } })

// Mayor o igual
db.productos.find({ precio: { $gte: 100 } })

// Menor o igual
db.productos.find({ precio: { $lte: 50 } })

// No igual
db.productos.find({ categoria: { $ne: "electronica" } })

// En lista
db.productos.find({ categoria: { $in: ["electronica", "hogar"] } })

// No en lista
db.productos.find({ categoria: { $nin: ["ropa", "alimentacion"] } })
```

Los operadores de comparacion pueden combinarse en el mismo campo:

```javascript
db.productos.find({ precio: { $gte: 50, $lte: 200 } })
```

## Operadores logicos

Para combinar multiples condiciones, MongoDB ofrece operadores logicos de primer nivel.

```javascript
// AND implicito (por defecto)
db.productos.find({ categoria: "electronica", precio: { $lt: 100 } })

// AND explicito
db.productos.find({
  $and: [
    { precio: { $gte: 50 } },
    { precio: { $lte: 200 } }
  ]
})

// OR
db.productos.find({
  $or: [
    { categoria: "electronica" },
    { precio: { $lt: 20 } }
  ]
})

// NOR (ni una condicion ni la otra)
db.productos.find({
  $nor: [
    { categoria: "electronica" },
    { precio: { $gte: 100 } }
  ]
})

// NOT (negacion de una condicion)
db.productos.find({ precio: { $not: { $gte: 100 } } })
```

El AND implicito funciona cuando las condiciones son sobre campos distintos. Si se necesita aplicar multiples condiciones sobre el mismo campo (por ejemplo, rango de precios), se usa un unico documento con operadores. El AND explicito es util cuando las condiciones involucran el mismo campo de forma no trivial.

## Consultas en campos anidados

Para acceder a campos dentro de documentos embebidos se usa la notacion de punto como string en la clave del filtro:

```javascript
db.usuarios.find({ "direccion.ciudad": "Madrid" })
db.usuarios.find({ "direccion.codigo_postal": { $regex: "^28" } })
```

Tambien puede usarse la notacion de punto para acceder a elementos especificos de un array por su indice:

```javascript
db.usuarios.find({ "tags.0": "importante" })
```

Para documentos embebidos completos se puede pasar un documento como valor, pero debe coincidir exactamente, incluido el orden de los campos:

```javascript
db.usuarios.find({ direccion: { ciudad: "Madrid", calle: "Mayor" } })
```

> La notacion de punto es preferible a pasar el documento completo porque es independiente del orden de los campos y permite usar operadores sobre subcampos concretos.

## Consultas en arrays

MongoDB ofrece varios operadores especificos para trabajar con arrays.

### Coincidencia exacta

Si se pasa un array como valor, MongoDB busca documentos que tengan exactamente ese array (mismos elementos en mismo orden):

```javascript
db.productos.find({ etiquetas: ["oferta", "nuevo"] })
```

Para buscar documentos donde un array contiene un elemento concreto (sin importar posicion ni otros elementos), se pasa el valor directamente:

```javascript
db.productos.find({ etiquetas: "oferta" })
```

### $all

`$all` selecciona documentos cuyo array contiene todos los elementos especificados, sin importar el orden:

```javascript
db.productos.find({ etiquetas: { $all: ["oferta", "nuevo"] } })
```

Equivale a un AND sobre elementos individuales pero en una sola condicion.

### $elemMatch

`$elemMatch` selecciona documentos donde al menos un elemento del array cumple todas las condiciones especificadas. Es util cuando se necesita que las condiciones se cumplan sobre el mismo elemento del array:

```javascript
db.pedidos.find({
  items: {
    $elemMatch: { producto: "laptop", cantidad: { $gte: 2 } }
  }
})
```

Sin `$elemMatch`, la condicion `{ "items.producto": "laptop", "items.cantidad": { $gte: 2 } }` podria coincidir con un documento donde un elemento tiene el producto "laptop" y otro elemento diferente tiene cantidad >= 2.

### $size

`$size` selecciona documentos cuyo array tiene una longitud exacta:

```javascript
db.productos.find({ etiquetas: { $size: 3 } })
```

> `$size` no acepta rangos. No se puede hacer `{ $size: { $gte: 2 } }`. Para eso se necesita un campo adicional que almacene la longitud o usar `$expr` con `$gte: [{ $size: "$etiquetas" }, 2]`.

## Proyeccion

La proyeccion permite controlar que campos se incluyen en los documentos devueltos. Es util para reducir el trafico de red y mejorar el rendimiento cuando solo se necesitan ciertos campos.

```javascript
// Incluir solo nombre y email (el _id se incluye por defecto)
db.usuarios.find({}, { nombre: 1, email: 1 })

// Excluir campos
db.usuarios.find({}, { password: 0, tarjeta: 0 })

// Excluir _id explicitamente
db.usuarios.find({}, { nombre: 1, _id: 0 })
```

No se pueden mezclar inclusiones y exclusiones excepto para el campo `_id`. La unica excepcion es que si se incluyen campos explicitamente (`{ campo: 1 }`), solo esos campos y `_id` se devuelven. Si se excluyen (`{ campo: 0 }`), se devuelven todos los demas.

## Paginacion

La paginacion en MongoDB se construye combinando tres metodos del cursor: `sort`, `skip` y `limit`.

### sort

Ordena los resultados por uno o varios campos:

```javascript
db.productos.find().sort({ precio: 1 })    // ascendente
db.productos.find().sort({ precio: -1 })   // descendente
db.productos.find().sort({ categoria: 1, precio: -1 })  // multi-campo
```

### skip y limit

`skip` salta un numero de documentos, `limit` limita el numero de documentos devueltos:

```javascript
db.productos.find()
  .sort({ precio: -1 })
  .skip(10)
  .limit(10)
```

Esta combinacion es la base de la paginacion offset-based (pagina 2, 10 elementos por pagina). Sin embargo, `skip` se vuelve ineficiente con valores grandes porque MongoDB debe escanear y descartar todos los documentos saltados.

> Para paginacion eficiente en colecciones grandes se recomienda el seek method (cursor-based pagination): usar `_id` o un campo indexado como referencia, combinando `{ _id: { $gt: lastSeenId } }` con `limit`.

### countDocuments

Para obtener el numero de documentos que coinciden con una consulta:

```javascript
db.usuarios.countDocuments({ activo: true })
```

`countDocuments` aplica el filtro y cuenta los documentos resultantes. Es preciso pero mas lento que `estimatedDocumentCount`, que devuelve una estimacion basada en metadatos de la coleccion sin aplicar filtro:

```javascript
db.usuarios.estimatedDocumentCount()
```

> `countDocuments` es equivalente a `find(query).count()` pero se recomienda su uso porque maneja correctamente los metadatos de la coleccion y las transacciones.

