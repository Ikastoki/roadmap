# Mantenimiento de indices

Los indices no son estructuras estaticas. Con el tiempo, las inserciones, actualizaciones y eliminaciones continuas fragmentan el arbol B-Tree, degradan el rendimiento y dejan indices que ya no se utilizan. El mantenimiento periodico permite recuperar espacio, mejorar la eficiencia de las consultas y eliminar indices obsoletos.

## reIndex

`reIndex()` elimina y reconstruye todos los indices de una coleccion desde cero. Es la operacion mas agresiva de mantenimiento y debe reservarse para situaciones donde se sospecha corrupcion o fragmentacion severa.

```javascript
db.ventas.reIndex()
```

Internamente, MongoDB descarta cada indice y lo recrea utilizando la misma especificacion, recorriendo todos los documentos de la coleccion para construir el arbol B-Tree desde cero.

Cuando usar `reIndex`:

- Despues de restaurar una coleccion desde un backup via `mongorestore`.
- Cuando el tamano en disco de los indices es significativamente mayor de lo esperado.
- Cuando `explain()` muestra planes de ejecucion anomalos que sugieren corrupcion de indice.
- Despues de importaciones masivas de datos.

Cuando NO usar `reIndex`:

- En horario de produccion sin ventana de mantenimiento. La operacion bloquea la coleccion durante todo el proceso.
- En colecciones con millones de documentos, donde puede tardar horas.
- Como mantenimiento preventivo rutinario (en su lugar usar `compact`).

> A partir de MongoDB 4.4, `reIndex()` ya no elimina y recrea cada indice secuencialmente en la mayoria de los casos, sino que utiliza `compact` internamente cuando es posible, reduciendo el tiempo de bloqueo.

## compact

`compact` reescribe los datos y los indices de una coleccion para eliminar fragmentacion interna y recuperar espacio no utilizado. A diferencia de `reIndex`, no reconstruye los indices desde cero, sino que compacta el arbol B-Tree existente.

```javascript
// Compactar una coleccion (requiere privilegios de administrador)
db.runCommand({ compact: "ventas" })
```

Opciones de `compact`:

- `force`: permite ejecutar `compact` en un replica set secundario (a partir de MongoDB 4.4).
- `comment`: identificador para auditoria.

El comportamiento exacto depende del motor de almacenamiento:

- **WiredTiger**: libera espacio no utilizado en disco y compacta los archivos del indice. Es una operacion que puede ejecutarse en linea, aunque con impacto en el rendimiento.
- **MMAPv1** (obsoleto): requiere detener las escrituras durante la operacion.

> En WiredTiger, `compact` no siempre reduce el tamano en disco. Solo recupera espacio si los archivos subyacentes tienen bloques vacios al final. MongoDB recomienda dejar que WiredTiger gestione la compactacion internamente y solo intervenir en casos extremos.

## $indexStats (estadisticas de uso)

El pipeline `$indexStats` proporciona metricas de uso de cada indice en una coleccion. Es la herramienta principal para identificar indices que no se estan utilizando.

```javascript
db.ventas.aggregate([{ $indexStats: {} }])
```

Salida tipica:

```json
[
  {
    "name": "_id_",
    "key": { "_id": 1 },
    "host": "servidor:27017",
    "accesses": { "ops": 1500000, "since": ISODate("2026-01-01T00:00:00Z") }
  },
  {
    "name": "idx_email",
    "key": { "email": 1 },
    "host": "servidor:27017",
    "accesses": { "ops": 5, "since": ISODate("2026-01-01T00:00:00Z") }
  }
]
```

Campos de interes:

- `accesses.ops`: numero de operaciones que han utilizado el indice.
- `accesses.since`: momento desde el que se contabilizan las operaciones (se reinicia al reiniciar mongod).

Los valores se acumulan desde el ultimo arranque del servidor. Para evaluar si un indice es util:

1. Ejecutar `$indexStats` tras un periodo representativo de actividad (varios dias o semanas).
2. Identificar indices con `ops` muy bajo o cero.
3. Verificar si esos indices se usan en algun momento del dia (consultas nocturnas, batch).
4. Revisar en los logs del profiler si alguna consulta depende de ese indice.

> Los indices con `ops` proximo a cero son candidatos a eliminacion. Sin embargo, hay que considerar consultas estacionales o batch que no se hayan ejecutado desde el ultimo reinicio.

## Eliminar indices no usados

Una vez identificados los indices candidatos con `$indexStats`, el proceso de eliminacion seguro sigue estos pasos:

1. **Documentar el indice**: guardar la especificacion en caso de necesitar recrearlo.
2. **Eliminar con `dropIndex()`**: ejecutar fuera del horario punta para minimizar impacto.
3. **Monitorizar rendimiento**: durante los dias siguientes verificar que no aparecen consultas lentas nuevas.
4. **Analizar logs del profiler**: confirmar que ninguna consulta dependia del indice eliminado.

```javascript
// Paso 1: obtener especificacion antes de eliminar
db.ventas.getIndexes()

// Paso 2: eliminar el indice
db.ventas.dropIndex("idx_campo_inutil")

// Paso 3: verificar que ya no aparece en la lista
db.ventas.getIndexes()
```

Si tras eliminar el indice se detecta que alguna consulta se ha vuelto lenta, hay que recrearlo inmediatamente usando la especificacion documentada.

> Mantener indices no usados tiene un coste real: ocupan RAM en el cache de WiredTiger, ralentizan las escrituras y aumentan el tiempo de sincronizacion en replica sets. Una limpieza trimestral de indices es una practica recomendada en entornos de produccion.

