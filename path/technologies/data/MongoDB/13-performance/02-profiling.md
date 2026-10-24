# Profiling

El profiler de MongoDB es un mecanismo integrado que registra las operaciones que superan ciertos umbrales de duracion en la coleccion `system.profile`. A diferencia de las herramientas de monitorizacion externas, el profiler captura la actividad interna del servidor con el contexto completo de cada operacion: la consulta exacta, los planes de ejecucion, los tiempos de bloqueo y los escaneos realizados.

## Niveles de profiling

El profiler tiene tres niveles de operacion:

- **Nivel 0**: desactivado. No se registra ninguna operacion. Es el valor por defecto en instalaciones nuevas.
- **Nivel 1**: registra solo las operaciones lentas, definidas por `slowOpThresholdMs`. Es el nivel recomendado para produccion, ya que captura las consultas problematica sin generar ruido ni sobrecarga apreciable.
- **Nivel 2**: registra todas las operaciones, independientemente de su duracion. Util en entornos de desarrollo o depuracion puntual, pero en produccion puede llenar rapidamente `system.profile` y afectar al rendimiento por la sobrecarga de escritura.

```javascript
// Ver el nivel actual
db.getProfilingLevel()

// Ver el nivel y el umbral
db.getProfilingStatus()
```

## Configuracion

`db.setProfilingLevel()` permite cambiar el nivel y el umbral de operacion lenta en milisegundos.

```javascript
// Nivel 1 con umbral de 100ms
db.setProfilingLevel(1, 100)

// Nivel 2 (todas las operaciones)
db.setProfilingLevel(2)

// Nivel 0 (desactivar)
db.setProfilingLevel(0)
```

El parametro `slowOpThresholdMs` se puede definir globalmente o por base de datos. En MongoDB 4.0 y superiores, tambien es posible definirlo por operacion en segundo plano (por ejemplo, `db.currentOp()` acepta un filtro de `secs_running`).

> El profiler consume recursos de escritura. En nivel 1 con un umbral razonable (100-200ms) el impacto es minimo. En nivel 2 el volumen de registros puede ser enorme incluso en bases de datos poco activas.

## La coleccion system.profile

Cuando el profiler esta activo, MongoDB escribe los registros en la coleccion especial `system.profile` de la base de datos actual. Esta coleccion tiene un tamano fijo (capped collection) de 1 MB por defecto, aunque puede redimensionarse.

```javascript
// Consultar las ultimas operaciones lentas
db.system.profile.find().sort({ ts: -1 }).limit(5).pretty()
```

Cada documento en `system.profile` contiene:

- `op`: tipo de operacion (insert, query, update, remove, getmore, command).
- `ns`: coleccion sobre la que se ejecuto.
- `query` o `command`: la consulta o comando exacto.
- `responseLength`: bytes devueltos.
- `millis`: duracion en milisegundos.
- `planSummary`: COLLSCAN, IXSCAN, etc.
- `execStats`: estadisticas detalladas de ejecucion, equivalentes a las de `explain("executionStats")`.
- `ts`: timestamp de la operacion.
- `client`: direccion del cliente que origino la operacion.

```javascript
// Buscar operaciones lentas con COLLSCAN
db.system.profile.find({
  millis: { $gt: 200 },
  planSummary: "COLLSCAN"
}).sort({ ts: -1 }).limit(10)
```

La informacion de `execStats` permite identificar exactamente cuantos documentos se examinaron (`totalDocsExamined`) frente a cuantos se devolvieron (`nReturned`), que indices se usaron y cuantas etapas de agregacion se ejecutaron.

### Tamano de system.profile

Por defecto, `system.profile` es una coleccion capped de 1 MB, lo que significa que cuando se llena, los documentos mas antiguos se sobrescriben. En sistemas con muchas operaciones lentas, 1 MB puede no ser suficiente para conservar un historial util. Es posible crear una coleccion `system.profile` con mayor capacidad antes de activar el profiler.

```javascript
// Crear system.profile con 10 MB
db.createCollection("system.profile", {
  capped: true,
  size: 10485760  // 10 MB
})

// Redimensionar (solo desactivando el profiler primero)
db.setProfilingLevel(0)
db.system.profile.drop()
db.createCollection("system.profile", { capped: true, size: 10485760 })
db.setProfilingLevel(1, 100)
```

> En entornos de produccion con muchas operaciones lentas, es recomendable usar una herramienta externa (Prometheus, Datadog, MongoDB Atlas) en lugar de depender exclusivamente de `system.profile`. La coleccion capped puede sobrescribir registros antes de que se analicen.

## Estrategia de uso en produccion

El profiler no debe estar permanentemente en nivel 2 en produccion. La estrategia comun:

1. Mantener nivel 1 con un umbral de 100ms para capturar consultas lentas sin ruido.
2. Ante un incidente de rendimiento, subir temporalmente a nivel 2 sobre la base de datos afectada durante unos minutos para capturar todas las operaciones.
3. Analizar los registros de `system.profile` para identificar patrones, consultas sin indice y operaciones con escaneos excesivos.
4. Volver a nivel 1 tras el analisis.

```javascript
// Activacion temporal para depuracion
db.setProfilingLevel(2)
// ... esperar 5 minutos ...
db.system.profile.find().sort({ ts: -1 }).limit(50).pretty()
// Restaurar nivel 1
db.setProfilingLevel(1, 100)
```

