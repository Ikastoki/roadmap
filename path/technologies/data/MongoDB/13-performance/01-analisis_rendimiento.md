# Analisis de rendimiento

El rendimiento de MongoDB depende de como interactuan los datos, la memoria, el disco y la concurrencia. Sin herramientas de diagnostico, cualquier decision de optimizacion es especulativa. MongoDB proporciona un conjunto de comandos y metodos que exponen metricas internas del motor de almacenamiento, el planificador de consultas, el estado de las operaciones activas y el estado global del servidor. Entender cada uno de ellos permite identificar cuellos de botella antes de que afecten a produccion.

## Estadisticas de base de datos y colecciones

`db.stats()` devuelve un resumen de la base de datos actual: tamanio total de datos, cantidad de objetos, indice promedio por documento, espacio en disco utilizado y fragmentacion. Es util para responder preguntas como "cuanto espacio ocupa esta base de datos" o "cuantos documentos tengo en total".

```javascript
db.stats()
```

La salida incluye campos como `dataSize` (tamanio real de los documentos sin compresion), `storageSize` (tamanio en disco tras compresion WiredTiger), `indexSize` (suma de todos los indices) y `indexes` (numero total de indices). Cuando `storageSize` es significativamente menor que `dataSize`, la compresion de WiredTiger esta funcionando eficazmente.

`db.collection.stats()` opera al nivel de una coleccion concreta e incluye informacion adicional como el numero de fragmentos extents (en MMAPv1, ya obsoleto) o el arbol B-tree de indices en WiredTiger.

```javascript
db.pedidos.stats()
```

Los campos relevantes para rendimiento son:

- `avgObjSize`: tamanio medio de cada documento. Valores que crecen con el tiempo sugieren que los documentos se estan actualizando con campos nuevos sin una politica de crecimiento controlado.
- `wiredTiger.cache.[read/write] bytes`: volumen de datos que pasa por la cache. Si `read bytes` supera el tamano de la cache repetidamente, el working set no cabe en RAM.
- `totalIndexSize`: suma del tamano de todos los indices. Si se acerca a `dataSize`, los indices consumen tanto espacio como los datos, lo que puede ralentizar las escrituras.

## Explain

`explain()` es la herramienta mas poderosa para depurar el rendimiento de consultas individuales. Acepta tres modos:

- `"queryPlanner"`: muestra el plan elegido sin ejecutar la consulta. Rapido, pero no da metricas reales de ejecucion.
- `"executionStats"`: ejecuta la consulta y devuelve estadisticas detalladas. Es el modo que se usa en la practica.
- `"allPlansExecution"`: ejecuta todos los planes considerados y los compara. Util para entender por que el optimizador descarto ciertos caminos.

```javascript
db.pedidos.find({ cliente: "ABC", total: { $gt: 100 } }).explain("executionStats")
```

### COLLSCAN vs IXSCAN

La salida de `explain()` revela el tipo de acceso a los datos en `stage`. Los dos valores fundamentales son:

- `COLLSCAN`: escaneo completo de la coleccion. MongoDB recorre todos los documentos uno por uno. Es inevitable cuando no existe un indice adecuado, pero en colecciones grandes supone decenas o cientos de miles de lecturas de disco.
- `IXSCAN`: escaneo de indice. MongoDB recorre las entradas del indice y solo recupera los documentos que cumplen la condicion. Es drasticamente mas eficiente, especialmente cuando la selectividad es alta.

`totalDocsExamined` indica cuantos documentos ha leido MongoDB para resolver la consulta, mientras que `nReturned` es el numero de documentos devueltos al cliente. La relacion `totalDocsExamined / nReturned` es el factor de sobreseleccion. Un valor cercano a 1 indica que la consulta esta perfectamente afinada. Un valor de 10000 significa que MongoDB examina diez mil documentos por cada documento devuelto.

> Un `IXSCAN` no garantiza eficiencia por si solo. Si el indice tiene baja selectividad (muchos documentos por clave), `totalDocsExamined` seguira siendo alto. La combinacion de indice correcto, filtro selectivo y proyeccion limitada es lo que produce consultas rapidas.

```javascript
// Ejemplo de salida relevante de explain()
// "stage": "IXSCAN",
// "totalDocsExamined": 150,
// "nReturned": 145,
// "executionTimeMillis": 12
```

## Operaciones activas

`db.currentOp()` lista todas las operaciones que se estan ejecutando en el servidor en ese momento. Es el equivalente de `SHOW PROCESSLIST` en MySQL.

```javascript
db.currentOp()
```

Filtra por estado, duracion, operacion o coleccion para localizar consultas lentas:

```javascript
db.currentOp({
  "active": true,
  "secs_running": { "$gt": 5 },
  "op": "query"
})
```

Los campos clave en cada operacion son:

- `secs_running`: tiempo que lleva ejecutandose. Operaciones que superan varios segundos en colecciones pequenas indican falta de indice o bloqueo.
- `ns`: coleccion sobre la que opera.
- `command`: la consulta exacta que se esta ejecutando.
- `planSummary`: resumen del plan de ejecucion (ej. `COLLSCAN`).
- `waitingForLock`: si es `true`, la operacion esta bloqueada esperando un recurso.

Cuando una consulta lleva demasiado tiempo, `db.killOp(opid)` permite terminarla sin reiniciar el servidor.

```javascript
db.killOp(12345)
```

## Estado del servidor

`db.serverStatus()` devuelve cientos de metricas sobre el estado actual del servidor: conexiones activas, uso de memoria, operaciones por segundo, trafico de red, actividad de la cache de WiredTiger, replicacion y mucho mas.

```javascript
db.serverStatus()
```

Las secciones mas relevantes para rendimiento son:

- `connections.current` y `connections.available`: numero de conexiones activas frente al maximo configurado. Si `current` se acerca a `available`, la aplicacion esta agotando el pool de conexiones.
- `opcounters`: total de operaciones de cada tipo (insert, query, update, delete, command). Permite calcular la tasa de operaciones por segundo si se muestrea en el tiempo.
- `network.bytesIn` y `network.bytesOut`: volumen de trafico. Un `bytesOut` alto combinado con `nReturned` alto sugiere que las consultas devuelven demasiados datos (falta de proyeccion).
- `wiredTiger.cache.tracked dirty bytes in the cache`: si este numero es persistentemente alto, la escritura en disco no sigue el ritmo de las escrituras en RAM.
- `wiredTiger.concurrentTransactions.readInto` y `writeInto`: tiquets de lectura y escritura disponibles. Si hay largas colas (read/write que esperan), WiredTiger esta saturado por concurrencia.
- `mem.resident` y `mem.virtual`: memoria fisica y virtual usada. Si `resident` supera la RAM disponible, el sistema comienza a hacer swap.

> `db.serverStatus()` no debe ejecutarse en produccion con alta frecuencia. Devuelve decenas de kilobytes y calcularlo tiene un coste, aunque pequeno. Para monitorizacion continua es preferible usar MongoDB Cloud Manager, Ops Manager o herramientas de terceros.

