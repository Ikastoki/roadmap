# Actualizar documentos

MongoDB ofrece varios metodos para modificar documentos existentes, desde actualizaciones puntuales de campos hasta el reemplazo completo del documento. A diferencia de SQL, donde `UPDATE` puede cambiar cualquier columna, MongoDB requiere operadores especificos para indicar que tipo de modificacion se desea realizar.

## updateOne

`updateOne` modifica el primer documento que coincide con el filtro. Requiere al menos un operador de actualizacion.

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $set: { activo: false } }
)
```

El resultado indica cuantos documentos coincidieron con el filtro (`matchedCount`) y cuantos se modificaron realmente (`modifiedCount`):

```javascript
{ acknowledged: true, matchedCount: 1, modifiedCount: 1 }
```

Si `matchedCount` es 0 y `upsert` no esta habilitado, `modifiedCount` sera 0.

## updateMany

`updateMany` aplica la misma modificacion a todos los documentos que coinciden con el filtro:

```javascript
db.productos.updateMany(
  { precio: { $lt: 10 } },
  { $set: { descatalogado: true } }
)
```

Es equivalente a ejecutar `updateOne` repetidamente, pero en una sola operacion atomica a nivel de servidor.

## replaceOne

`replaceOne` reemplaza completamente el contenido de un documento, excepto el campo `_id` que permanece intacto:

```javascript
db.usuarios.replaceOne(
  { nombre: "Ana" },
  { nombre: "Ana Garcia", email: "ana.garcia@example.com", activo: true }
)
```

> `replaceOne` no usa operadores. El segundo argumento es el documento de reemplazo completo. Si se omite un campo, ese campo desaparece del documento. Es equivalente a borrar el documento e insertar uno nuevo con el mismo `_id`.

## Operadores de actualizacion

### $set

Establece el valor de uno o varios campos. Si el campo no existe, lo crea:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $set: { email: "ana.nuevo@example.com", telefono: "123456789" } }
)
```

### $unset

Elimina un campo del documento:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $unset: { telefono: "" } }
)
```

El valor asociado a `$unset` es irrelevante; se usa comunmente `""` o `1`.

### $inc

Incrementa (o decrementa) un campo numerico en la cantidad especificada:

```javascript
db.productos.updateOne(
  { nombre: "laptop" },
  { $inc: { stock: -1, vendidos: 1 } }
)
```

Si el campo no existe, `$inc` lo crea con el valor especificado.

### $mul

Multiplica un campo numerico por el valor indicado:

```javascript
db.productos.updateOne(
  { nombre: "laptop" },
  { $mul: { precio: 0.9 } }  // 10% de descuento
)
```

### $rename

Renombra un campo:

```javascript
db.usuarios.updateMany(
  {},
  { $rename: { "nombre_completo": "nombre" } }
)
```

### $min y $max

`$min` actualiza el campo solo si el nuevo valor es menor que el actual. `$max` solo si es mayor:

```javascript
db.productos.updateOne(
  { nombre: "laptop" },
  { $min: { precio: 800 } }  // solo si precio > 800
)

db.productos.updateOne(
  { nombre: "laptop" },
  { $max: { precio: 1200 } } // solo si precio < 1200
)
```

Son utiles para mantener limites inferiores o superiores sin necesidad de leer el documento primero.

### $currentDate

Establece un campo a la fecha actual del servidor:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $currentDate: { ultimo_acceso: true } }
)
```

Puede usar `{ $type: "date" }` o `{ $type: "timestamp" }` para controlar el tipo de dato.

## Operadores para arrays

### $push

Agrega un elemento al final del array:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $push: { etiquetas: "premium" } }
)
```

Para agregar multiples elementos se usa `$each`:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $push: { etiquetas: { $each: ["vip", "frecuente", "descuento"] } } }
)
```

### $pull

Elimina del array todos los elementos que coinciden con la condicion:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $pull: { etiquetas: "antiguo" } }
)

// Con condicion
db.productos.updateOne(
  { nombre: "laptop" },
  { $pull: { comentarios: { puntuacion: { $lt: 3 } } } }
)
```

### $addToSet

Agrega un elemento al array solo si no existe ya:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $addToSet: { etiquetas: "premium" } }
)
```

Si "premium" ya existe en el array, no se realiza ninguna modificacion. Puede combinarse con `$each` para agregar varios elementos sin duplicados:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $addToSet: { etiquetas: { $each: ["nuevo", "destacado"] } } }
)
```

### $pop

Elimina el primer (`-1`) o ultimo (`1`) elemento del array:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  { $pop: { notificaciones: -1 } }  // elimina la mas antigua
)
```

### $slice

`$slice` se usa en combinacion con `$push` para limitar el tamano maximo del array. Cuando se usa `$each` con `$slice`, el array se trunca a la longitud especificada tras la insercion:

```javascript
db.usuarios.updateOne(
  { nombre: "Ana" },
  {
    $push: {
      notificaciones: {
        $each: [{ mensaje: "nueva notificacion", fecha: new Date() }],
        $slice: -10   // mantiene solo las 10 mas recientes
      }
    }
  }
)
```

### $[] (todos los elementos)

El operador posicional `$[]` actualiza todos los elementos de un array sin necesidad de conocer sus indices:

```javascript
db.pedidos.updateOne(
  { _id: 1 },
  { $set: { "items.$[].estado": "procesado" } }
)
```

Esto establece `estado: "procesado"` en todos los elementos del array `items`.

### $[identifier] y arrayFilters

El operador posicional filtrado `$[<identifier>]` actualiza solo los elementos del array que cumplen una condicion definida en `arrayFilters`:

```javascript
db.pedidos.updateOne(
  { _id: 1 },
  { $set: { "items.$[elem].precio": 0 } },
  {
    arrayFilters: [
      { "elem.producto": "laptop", "elem.cantidad": { $gte: 2 } }
    ]
  }
)
```

Este patron permite actualizaciones selectivas dentro de arrays sin conocer la posicion exacta de los elementos.

## Upsert

`upsert: true` convierte la operacion de actualizacion en un "inserta si no existe". Si el filtro no encuentra documentos coincidentes, se inserta un nuevo documento combinando el filtro con la actualizacion:

```javascript
db.contadores.updateOne(
  { pagina: "inicio" },
  { $inc: { visitas: 1 } },
  { upsert: true }
)
```

Si no existe un documento con `pagina: "inicio"`, MongoDB crea uno con `{ pagina: "inicio", visitas: 1 }`. Si existe, simplemente incrementa `visitas`.

> Con `upsert: true`, `matchedCount` puede ser 0 pero `upsertedCount` sera 1 y el resultado incluira el `_id` del documento insertado.

