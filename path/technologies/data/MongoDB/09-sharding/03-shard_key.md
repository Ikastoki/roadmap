# Eleccion de la shard key

La shard key es el criterio que determina como se distribuyen los documentos entre los shards. Una eleccion correcta garantiza que el cluster escale de forma eficiente; una eleccion erronea puede degradar el rendimiento hasta hacer el sistema inutilizable. MongoDB no permite cambiar la shard key una vez que la coleccion ha sido shardeada, por lo que esta decision debe tomarse conociendo a fondo el patron de acceso a los datos.

## Propiedades de una buena shard key

Toda shard key debe satisfacer tres requisitos para que la distribucion sea efectiva:

**Alta cardinalidad.** La shard key debe tener tantos valores distintos como sea posible. Una shard key con pocos valores (por ejemplo, un booleano con solo `true` y `false`) solo permite crear dos chunks, lo que limita el numero maximo de shards que pueden participar en la distribucion. Cuanto mayor sea la cardinalidad, mas granular sera el reparto y mejor se aprovecharan los recursos del cluster.

**Baja frecuencia de cambio.** Cuando el valor de la shard key de un documento cambia, MongoDB puede necesitar migrar el chunk completo a otro shard, lo que implica movimiento de datos y sobrecarga en la red. Lo ideal es que la shard key sea un campo que se escriba una sola vez, como un identificador unico.

**Distribucion uniforme.** La shard key debe generar valores que se repartan de forma homogenea entre todos los shards. Si un valor aparece con mucha mas frecuencia que los demas, el chunk que lo contiene crecera desproporcionadamente y el shard correspondiente recibira mas carga.

## Hashed vs Ranged

La eleccion entre hashed y ranged no es una cuestion de cual es mejor en abstracto, sino de cual se adapta mejor al patron de consultas de la aplicacion.

### Cuando usar hashed

El hashed sharding es la opcion mas segura cuando no existe un patron de consultas claro o cuando las escrituras son el cuello de botella principal. Al distribuir los valores mediante una funcion hash, MongoDB garantiza que los documentos se repartan uniformemente entre los shards.

```javascript
sh.shardCollection("midb.eventos", { _id: "hashed" })
```

Es especialmente util para:
- Colecciones donde la shard key natural tiene baja cardinalidad.
- Sistemas con alta tasa de escrituras que necesitan distribuir la carga uniformemente.
- Campos monotonicos como timestamps o autoincrementales.

La contrapartida es que las consultas por rango sobre la shard key pierden la capacidad de ser dirigidas. Una consulta como `db.eventos.find({ _id: { $gt: ObjectId("...") } })` debera enviarse a todos los shards.

### Cuando usar ranged

El ranged sharding es superior cuando las consultas principales operan por rangos de valores. Una aplicacion que busca eventos entre dos fechas, o usuarios cuyos nombres estan en un rango alfabetico, se beneficia de que mongos pueda enviar la consulta solo al shard o shards que contienen ese rango.

```javascript
sh.shardCollection("midb.eventos", { fecha: 1 })
```

El riesgo principal del ranged sharding es la distribucion desigual. Si la shard key es monotona (como una fecha), todas las escrituras se concentraran en el chunk del extremo superior, desaprovechando el resto de shards.

> En la practica, la mayoria de los clusters de produccion usan hashed sharding para colecciones con alta tasa de escritura y ranged sharding para colecciones donde predominan las consultas analiticas por rangos.

## Compound shard key

Cuando un solo campo no es suficiente para obtener alta cardinalidad y distribucion uniforme, se puede utilizar una shard key compuesta por dos o mas campos:

```javascript
sh.shardCollection("midb.pedidos", { cliente_id: 1, fecha: 1 })
```

La shard key compuesta permite:
- **Incrementar la cardinalidad**: si `cliente_id` tiene 10.000 valores y `fecha` tiene 365 valores al ano, la combinacion ofrece 3.650.000 valores posibles.
- **Mejorar la distribucion**: aunque `fecha` sea monotona, la combinacion con `cliente_id` hace que los valores se distribuyan entre todos los shards.
- **Soportar consultas por prefijo**: las consultas que incluyen `cliente_id` (el campo de mayor jerarquia) pueden ser dirigidas aunque no incluyan `fecha`.

> La compound shard key no tiene por que ser un indice existente. MongoDB crea automaticamente un indice compuesto sobre los campos de la shard key si no existe previamente.

## Problemas comunes con la shard key

Aunque la shard key se elija con cuidado, ciertos problemas pueden aparecer con el tiempo.

### Hotspots

Un hotspot es un shard que recibe significativamente mas carga que el resto. Ocurre cuando la shard key no distribuye los datos uniformemente. Por ejemplo, una shard key basada en `pais` en una aplicacion donde el 70% de los usuarios son de un mismo pais concentrara la mayoria de los documentos en el chunk de ese pais.

Los hotspots se mitigan con hashed sharding (si la consulta lo permite) o con una compound shard key que incluya un campo de alta cardinalidad como prefijo.

### Scatter-gather

Cuando una consulta no incluye la shard key, mongos no puede determinar que shard contiene los datos y debe enviar la consulta a todos los shards para despues combinar los resultados. Esto se conoce como scatter-gather y es drasticamente mas lento que una consulta dirigida.

```javascript
// Consulta dirigida: mongos sabe exactamente que shard consultar
db.pedidos.find({ cliente_id: "ABC123" })

// Scatter-gather: mongos debe preguntar a todos los shards
db.pedidos.find({ estado: "pendiente" })
```

Cuantas mas consultas de scatter-gather existan, mas se degrada el rendimiento del cluster. La solucion es disenar las consultas para que incluyan siempre la shard key, o bien crear un indice secundario en cada shard (aunque mongos no pueda dirigir la consulta).

### Jumbo chunks

Un jumbo chunk es un chunk que ha superado el tamano maximo configurado (por defecto 64 MB) pero que MongoDB no puede dividir porque todos los documentos comparten el mismo valor de shard key. Por ejemplo, si la shard key es `pais` y existen millones de documentos con `pais: "US"`, todos caeran en el mismo chunk y este nunca podra dividirse.

Los jumbo chunks:
- No pueden moverse a otro shard.
- Degradan el rendimiento del shard que los contiene.
- Hacen que el balancer no pueda equilibrar la carga.

> MongoDB no permite automaticamente la migracion de jumbo chunks. Para solucionarlos, a menudo es necesario intervenir manualmente: mover el chunk con `moveChunk` o, en casos extremos, exportar e importar los datos con una shard key diferente.

