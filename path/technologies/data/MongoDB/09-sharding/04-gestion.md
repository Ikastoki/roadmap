# Gestion del sharded cluster

Una vez el cluster shardeado esta en produccion, es necesario monitorizar su estado, supervisar la distribucion de los datos y controlar el comportamiento del balancer para mantener un rendimiento optimo. MongoDB proporciona comandos especificos para cada una de estas tareas.

## sh.status(): estado del cluster

El comando fundamental para conocer el estado del cluster es `sh.status()`. Su salida incluye informacion detallada sobre los shards registrados, las bases de datos habilitadas para sharding, las colecciones shardeadas y la distribucion de chunks:

```
sharding version: 1
  shards:
    { "_id": "shard1RS", "host": "shard1RS/shard1a:27018,shard1b:27018,shard1c:27018" }
    { "_id": "shard2RS", "host": "shard2RS/shard2a:27018,shard2b:27018,shard2c:27018" }
  databases:
    { "_id": "midb", "primary": "shard1RS", "partitioned": true }
      midb.pedidos
        shard key: { "cliente_id": 1, "fecha": 1 }
        chunks:
          shard1RS 3
          shard2RS 2
        { "cliente_id": { "$minKey": 1 }, "fecha": { "$minKey": 1 } } -->>
        { "cliente_id": "A", "fecha": { "$minKey": 1 } } on shard1RS
        { "cliente_id": "A", "fecha": { "$minKey": 1 } } -->>
        { "cliente_id": "M", "fecha": { "$minKey": 1 } } on shard2RS
```

La seccion de chunks muestra los rangos exactos que cubre cada chunk y el shard donde reside. Esta informacion es util para identificar desbalances o chunks excesivamente grandes.

Para obtener una version mas concisa y rapida, se puede utilizar `sh.status({ verbose: false })`.

## Distribucion de chunks

La distribucion de chunks entre shards deberia ser lo mas equilibrada posible. Si un shard acumula muchos mas chunks que los demas, las escrituras y lecturas se concentraran en el, anulando los beneficios del sharding.

Para ver cuantos chunks tiene cada shard en una coleccion especifica:

```javascript
db.getSiblingDB("config").collections.find({ _id: "midb.pedidos" }).pretty()
```

O, de forma mas directa, revisar la coleccion `config.chunks`:

```javascript
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "midb.pedidos" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])
```

Cuando la distribucion esta desequilibrada, se puede forzar el movimiento manual de un chunk con `sh.moveChunk()`:

```javascript
sh.moveChunk("midb.pedidos", { cliente_id: "M" }, "shard1RS")
```

> `sh.moveChunk()` debe usarse con cuidado: mover un chunk implica transferir todos sus documentos a otro shard, lo que genera carga en la red y en los discos.

## Gestion del balancer

El balancer es un proceso en segundo plano que se ejecuta en los config servers y se encarga de mantener la distribucion de chunks equilibrada entre los shards. Cuando detecta que la diferencia entre el shard con mas chunks y el que tiene menos supera un umbral (por defecto 8), inicia la migracion de chunks del shard mas cargado al menos cargado.

### Control del balancer

El balancer se enciende y apaga con comandos simples:

```javascript
// Detener el balancer
sh.stopBalancer()

// Reanudar el balancer
sh.startBalancer()
```

Detener el balancer es util durante ventanas de mantenimiento o cuando se espera un pico de carga y no se desea que las migraciones compitan por recursos de red y disco.

### Ventana de balanceo

Para evitar que el balancer opere en horas punta, se puede definir una ventana de tiempo en la que esta permitido ejecutar migraciones:

```javascript
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "02:00", end: "06:00" } } },
  { upsert: true }
)
```

Esta configuracion restringe el balanceo a las horas de menor actividad (de 2:00 a 6:00 de la manana), cuando el impacto en los usuarios es minimo.

### Estado del balancer

Para comprobar si el balancer esta activo y si hay migraciones en curso:

```javascript
sh.isBalancerRunning()  // devuelve true/false
```

## Chunk splitting

MongoDB divide automaticamente los chunks cuando superan el tamano maximo configurado (por defecto 64 MB). Este proceso se conoce como chunk splitting y es gestionado por el propio mongod sin intervencion manual.

Cuando un chunk alcanza el limite de tamano, MongoDB busca un punto de division dentro del rango del chunk basandose en los valores de la shard key. Si encuentra un valor que divida el chunk en dos mitades aproximadamente iguales, lo parte en dos chunks independientes.

> Si todos los documentos del chunk comparten el mismo valor de shard key, MongoDB no puede dividirlo y se forma un jumbo chunk. La unica forma de evitarlo es elegir una shard key con alta cardinalidad.

El umbral de 64 MB puede ajustarse globalmente, aunque no es recomendable modificarlo sin una razon de peso:

```javascript
db.getSiblingDB("config").settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 128 } },
  { upsert: true }
)
```

## Monitoreo continuo del cluster

Ademas de `sh.status()`, existen varias herramientas para monitorizar la salud del cluster:

- **mongostat**: muestra metricas de operaciones por segundo en cada shard.
- **mongotop**: indica el tiempo que cada coleccion dedica a lecturas y escrituras.
- **db.collection.stats()**: incluye informacion sobre el numero de chunks, el tamano total de la coleccion y la distribucion por shard.

```javascript
db.pedidos.stats({ scale: 1048576 })  // tamano en MB
```

La salida de `stats()` incluye el apartado `shards` con las metricas individuales de cada shard:

```json
"shards": {
  "shard1RS": { "count": 1500000, "size": 2048, "numChunks": 3 },
  "shard2RS": { "count": 1400000, "size": 1950, "numChunks": 2 }
}
```

> Un cluster shardeado requiere monitorizacion constante. La mayoria de los problemas de rendimiento en clusters grandes se detectan primero en la distribucion desigual de chunks o en la saturacion de un shard especifico.

