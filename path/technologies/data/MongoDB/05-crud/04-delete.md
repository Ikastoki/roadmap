# Eliminar documentos

MongoDB proporciona varios metodos para eliminar documentos, desde la eliminacion individual hasta el borrado completo de colecciones. La eliminacion en MongoDB es permanente: no existe un comando ROLLBACK ni una papelera de reciclaje, por lo que cualquier operacion de borrado debe ejecutarse con conocimiento de sus consecuencias.

## deleteOne

`deleteOne` elimina el primer documento que coincide con el filtro. Si hay multiples documentos que coinciden, solo se elimina el primero que encuentra MongoDB (tipicamente en orden natural de almacenamiento, no necesariamente el mas antiguo):

```javascript
db.usuarios.deleteOne({ nombre: "Ana" })
```

El resultado indica cuantos documentos se eliminaron:

```javascript
{ acknowledged: true, deletedCount: 1 }
```

Si ningun documento coincide, `deletedCount` sera 0.

## deleteMany

`deleteMany` elimina todos los documentos que coinciden con el filtro:

```javascript
db.usuarios.deleteMany({ activo: false })
```

Para eliminar todos los documentos de una coleccion sin borrar la coleccion en si:

```javascript
db.usuarios.deleteMany({})
```

`deleteMany({})` vacia la coleccion pero mantiene los indices, las reglas de validacion y otros metadatos. Es significativamente mas rapido que eliminar documentos uno por uno.

## findOneAndDelete

`findOneAndDelete` elimina un documento y lo devuelve en la misma operacion. Es util cuando se necesita el contenido del documento eliminado, por ejemplo para procesarlo antes de descartarlo o para implementar colas de trabajo:

```javascript
const eliminado = db.usuarios.findOneAndDelete({ nombre: "Ana" })
printjson(eliminado)
```

```javascript
{
  _id: ObjectId("665a..."),
  nombre: "Ana",
  email: "ana@example.com",
  activo: true
}
```

Acepta opciones adicionales como `sort` para controlar cual de los documentos coincidentes se elimina:

```javascript
const masAntiguo = db.cola.findOneAndDelete(
  { estado: "pendiente" },
  { sort: { creado_en: 1 } }
)
```

Esto elimina y devuelve el documento pendiente mas antiguo, patron tipico para implementar colas FIFO.

> `findOneAndDelete` es atomico a nivel de documento: no hay posibilidad de que otro cliente lea el documento entre la busqueda y la eliminacion.

## delete vs drop

La diferencia fundamental entre eliminar documentos y eliminar la coleccion completa tiene implicaciones operativas importantes.

**deleteMany({})**:
- Conserva la coleccion y toda su configuracion: indices, validaciones, reglas de collation.
- El espacio en disco ocupado por los documentos no se libera inmediatamente; WiredTiger lo reutiliza para nuevos documentos.
- Los indices se mantienen, aunque se marcan como vacios.
- Operacion que se registra en el oplog y puede replicarse.

**drop()**:
- Elimina la coleccion completa, incluidos todos sus documentos, indices y metadatos.
- Libera el espacio en disco inmediatamente.
- Es mas rapido que `deleteMany({})` en colecciones grandes.
- Cualquier operacion posterior que intente acceder a la coleccion debe crearla de nuevo (se crea implicitamente al insertar).
- Los indices deben recrearse explicitamente.

```javascript
// Eliminar todos los documentos
db.logs.deleteMany({})

// Eliminar la coleccion completa
db.logs.drop()
```

> `drop` no elimina la base de datos. Para eliminar la base de datos completa se usa `db.dropDatabase()`, que elimina la base de datos actual y todas sus colecciones.

La eleccion entre `deleteMany({})` y `drop` depende del caso de uso:
- Si se necesita mantener indices y configuracion (por ejemplo, en una coleccion temporal que se vacia periodicamente), usar `deleteMany({})`.
- Si se quiere liberar espacio en disco y la coleccion se va a recrear con indices diferentes, usar `drop`.

## Soft delete

El soft delete es un patron de diseno que evita la eliminacion fisica de documentos. En lugar de borrar un registro, se marca como eliminado mediante un campo que indica su estado. Esto permite mantener la integridad referencial, auditar eliminaciones y, si es necesario, revertirlas.

```javascript
// Marcar como eliminado (soft delete)
db.usuarios.updateOne(
  { _id: ObjectId("665a...") },
  {
    $set: {
      deletedAt: new Date(),
      deletedBy: "admin",
      deletedReason: "solicitud del usuario"
    }
  }
)

// Consultar solo documentos activos
db.usuarios.find({ deletedAt: null })

// Consultar documentos eliminados
db.usuarios.find({ deletedAt: { $ne: null } })

// Consultar todos (activos y eliminados)
db.usuarios.find({})
```

### Variantes del soft delete

**Campo booleano**: la implementacion mas simple usa un campo `activo` que se establece a `false` al eliminar:

```javascript
db.usuarios.updateOne(
  { _id: ObjectId("665a...") },
  { $set: { activo: false } }
)
```

**Campo deletedAt**: mas informativo que el booleano, almacena la fecha exacta de eliminacion y permite filtrar por fecha:

```javascript
db.usuarios.find({ deletedAt: { $gte: ISODate("2025-01-01") } })
```

**Documento de auditoria**: para casos donde se necesita conservar el historial completo, se mueve el documento eliminado a una coleccion separada de auditoria:

```javascript
const eliminado = db.usuarios.findOneAndDelete({ _id: ObjectId("665a...") })
db.usuarios_eliminados.insertOne({
  ...eliminado,
  deletedAt: new Date(),
  deletedBy: "admin"
})
```

### Indices para soft delete

Para que las consultas sobre documentos activos sean eficientes, se debe crear un indice parcial que solo indexe documentos no eliminados:

```javascript
db.usuarios.createIndex(
  { email: 1 },
  { partialFilterExpression: { deletedAt: null } }
)
```

Esto mantiene el indice mas pequeno y las consultas sobre documentos activos mas rapidas.

### Ventajas y desventajas

**Ventajas**:
- Los datos pueden restaurarse sin necesidad de backups.
- Se mantiene la integridad referencial si otras colecciones referencian estos documentos.
- Permite auditoria completa de eliminaciones.
- Facilita la implementacion de "papelera de reciclaje" con restauracion en la interfaz de usuario.

**Desventajas**:
- La coleccion crece con datos que logicamente ya no existen.
- Todas las consultas deben filtrar explicitamente los documentos eliminados.
- Si no se indexa correctamente, el rendimiento de las consultas se degrada.
- Los indices unicos pueden causar conflictos si se intenta recrear un registro "eliminado".

