# Gestion de indices

La gestion de indices abarca su creacion, eliminacion, inspeccion y analisis del plan de ejecucion. MongoDB proporciona comandos especificos para cada operacion, ademas de la capacidad de forzar un indice concreto cuando el planificador no escoge el optimo.

## createIndex y createIndexes

`createIndex` crea un indice en una coleccion. La sintaxis recibe el documento con la especificacion del indice y un documento opcional de opciones.

```javascript
db.usuarios.createIndex({ email: 1 })
db.usuarios.createIndex({ email: 1 }, { unique: true })
```

`createIndexes` permite crear varios indices en una sola operacion, lo que resulta mas eficiente que multiples llamadas a `createIndex` porque MongoDB solo sincroniza una vez los metadatos.

```javascript
db.usuarios.createIndexes([
  { key: { email: 1 }, name: "idx_email", unique: true },
  { key: { nombre: 1, apellido: 1 }, name: "idx_nombre_apellido" },
  { key: { createdAt: -1 }, name: "idx_createdAt", expireAfterSeconds: 86400 }
])
```

Opciones habituales en la creacion:

- `name`: nombre del indice (si no se especifica, MongoDB lo genera concatenando campo y direccion).
- `unique`: fuerza unicidad.
- `partialFilterExpression`: indice parcial.
- `sparse`: indice disperso.
- `expireAfterSeconds`: indice TTL.
- `background`: obsoleto desde MongoDB 4.2 (toda creacion es en background).

> Asignar nombres descriptivos a los indices facilita su identificacion en `getIndexes()` y `dropIndex()`. Por convencion se usa `idx_campo1_campo2`.

## dropIndex y dropIndexes

Eliminar un indice libera espacio en disco y reduce el coste de escritura. La eliminacion puede hacerse por nombre o por especificacion.

```javascript
// Por nombre
db.usuarios.dropIndex("idx_email")

// Por especificacion
db.usuarios.dropIndex({ email: 1 })
```

`dropIndexes` elimina todos los indices de la coleccion excepto `_id`:

```javascript
db.usuarios.dropIndexes()
```

No es posible eliminar el indice sobre `_id`. Intentarlo produce un error.

```javascript
db.usuarios.dropIndex({ _id: 1 })
// "cannot drop _id index"
```

> Eliminar un indice que esta siendo utilizado por consultas activas degrada inmediatamente el rendimiento. Conviene verificar el uso real con `$indexStats` antes de eliminar cualquier indice.

## getIndexes

Lista todos los indices de una coleccion con su nombre, campos, tipo y opciones.

```javascript
db.usuarios.getIndexes()
```

Salida tipica:

```json
[
  {
    "v": 2,
    "key": { "_id": 1 },
    "name": "_id_"
  },
  {
    "v": 2,
    "key": { "email": 1 },
    "name": "idx_email",
    "unique": true
  }
]
```

El campo `v` indica la version del formato del indice (actualmente 2 desde MongoDB 3.4). `key` contiene la especificacion del indice, y el resto de campos son opciones activas.

## explain

`explain()` es la herramienta fundamental para entender como se ejecuta una consulta. Acepta tres modos de detalle.

### queryPlanner (default)

Muestra el plan que el planificador considera optimo sin ejecutar la consulta.

```javascript
db.usuarios.find({ email: "user@example.com" }).explain("queryPlanner")
```

Campos relevantes en la salida:

- `winningPlan.stage`: `COLLSCAN`, `IXSCAN`, `FETCH`, `SHARD_MERGE`, etc.
- `winningPlan.inputStage`: etapa hija en planes anidados.
- `rejectedPlans`: planes que el planificador descarto.

### executionStats

Ejecuta la consulta y devuelve estadisticas detalladas.

```javascript
db.usuarios.find({ email: "user@example.com" }).explain("executionStats")
```

Campos clave:

- `executionTimeMillis`: tiempo real de ejecucion.
- `totalKeysExamined`: entradas del indice examinadas.
- `totalDocsExamined`: documentos de la coleccion examinados.
- `nReturned`: numero de documentos devueltos.
- `executionStages.stage`: plan de ejecucion.

La relacion entre `totalDocsExamined` y `nReturned` indica la eficiencia. Idealmente deben ser iguales. Si `totalDocsExamined` es muy superior a `nReturned`, el indice no esta filtrando bien.

```javascript
// Ejemplo optimo: covered query
// "totalDocsExamined": 0
// "totalKeysExamined": 1
// "nReturned": 1
```

### allPlansExecution

Ejecuta y compara todos los planes candidatos, no solo el ganador. Util para diagnosticar por que el planificador rechazo un plan que el desarrollador considera mejor.

```javascript
db.usuarios.find({ email: "user@example.com" }).explain("allPlansExecution")
```

> En produccion hay que usar `executionStats` con precaucion. Ejecutar la consulta para obtener estadisticas tiene coste. Para diagnosticar sin impacto usar `queryPlanner`.

## hint

`hint()` fuerza al planificador a utilizar un indice especifico, anulando su criterio. Util en escenarios donde el planificador escoge un plan suboptimo por errores en las estadisticas de cardinalidad.

```javascript
// Forzar uso del indice idx_email
db.usuarios.find({ email: "user@example.com" }).hint("idx_email")

// Tambien por especificacion
db.usuarios.find({ email: "user@example.com" }).hint({ email: 1 })
```

Tambien permite forzar un COLLSCAN explicitamente (util para comparar rendimiento):

```javascript
db.usuarios.find({ email: "user@example.com" }).hint({ $natural: 1 })
```

> `hint()` es una herramienta para diagnosticos y casos excepcionales. No debe usarse como practica habitual porque si el indice se elimina o renombra, la consulta fallara. Es mejor corregir el diseno del indice o las estadisticas que mantener hints permanentes.

