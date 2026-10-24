# Configuracion de un sharded cluster

Poner en marcha un cluster shardeado en MongoDB requiere configurar los tres componentes en el orden correcto: primero los config servers, despues los shards y finalmente los routers mongos. Una vez el cluster esta operativo, se habilitan las bases de datos y colecciones para sharding mediante comandos especificos.

## Arranque del cluster

### Config servers en replica set

Los config servers deben iniciarse como un replica set con nombre propio. En MongoDB 3.4 y posteriores se usa el modo `--configsvr` para indicar que un proceso mongod actuara como config server.

```bash
mongod --configsvr --replSet configRS --port 27019 --dbpath /data/configdb
```

Una vez iniciados los nodos, se conecta a uno de ellos y se inicializa el replica set:

```javascript
rs.initiate({
  _id: "configRS",
  configsvr: true,
  members: [
    { _id: 0, host: "config1:27019" },
    { _id: 1, host: "config2:27019" },
    { _id: 2, host: "config3:27019" }
  ]
})
```

### Shards como replica sets

Cada shard se configura como un replica set normal, sin parametros especiales:

```bash
mongod --replSet shard1RS --port 27018 --dbpath /data/shard1
```

Se inicializa el replica set de la misma forma que cualquier replica set:

```javascript
rs.initiate({
  _id: "shard1RS",
  members: [
    { _id: 0, host: "shard1a:27018" },
    { _id: 1, host: "shard1b:27018" },
    { _id: 2, host: "shard1c:27018" }
  ]
})
```

### Routers mongos

Con los config servers y shards operativos, se arrancan los procesos mongos indicando la cadena de conexion al replica set de configuracion:

```bash
mongos --configdb configRS/config1:27019,config2:27019,config3:27019 --port 27017
```

> mongos no necesita `--dbpath` porque no almacena datos: toda la configuracion se obtiene de los config servers en cada arranque.

Los shards se registran en el cluster desde el shell de mongos:

```javascript
sh.addShard("shard1RS/shard1a:27018,shard1b:27018,shard1c:27018")
sh.addShard("shard2RS/shard2a:27018,shard2b:27018,shard2c:27018")
sh.addShard("shard3RS/shard3a:27018,shard3b:27018,shard3c:27018")
```

## Habilitar sharding en una base de datos

El sharding no se activa por defecto. Es necesario habilitarlo a nivel de base de datos mediante `sh.enableSharding()`:

```javascript
sh.enableSharding("midb")
```

Este comando no distribuye los datos automaticamente; simplemente marca la base de datos como apta para sharding. Las colecciones dentro de ella no se shardearan hasta que se ejecute `sh.shardCollection()`.

> `sh.enableSharding()` solo debe ejecutarse una vez por base de datos. Intentar habilitarlo de nuevo no produce error pero tampoco tiene efecto.

## Shardear una coleccion

Una vez habilitada la base de datos, se shardea una coleccion especifica indicando la shard key:

```javascript
sh.shardCollection("midb.miColeccion", { campoShardKey: 1 })
```

El segundo parametro define la shard key y su direccion de ordenacion (1 para ascendente, -1 para descendente). MongoDB utilizara esta clave para particionar los documentos en chunks.

En el momento de shardear la coleccion, si ya contiene datos, MongoDB creara los chunks iniciales y los distribuira entre los shards disponibles. Si la coleccion esta vacia, MongoDB creara un unico chunk que crecera hasta alcanzar el tamano maximo antes de dividirse.

## Hashed vs Ranged: estrategias de sharding

La shard key puede configurarse con dos estrategias de distribucion que determinan como se agrupan los valores en chunks.

### Ranged sharding

La estrategia por defecto divide el espacio de valores de la shard key en rangos contiguos. Cada chunk cubre un intervalo de valores (por ejemplo, de "a" a "m" y de "n" a "z").

```javascript
sh.shardCollection("midb.usuarios", { nombre: 1 })
```

Ventajas:
- Las consultas por rango sobre la shard key son muy eficientes, porque mongos puede identificar exactamente que shards contienen los datos del rango solicitado.

Desventajas:
- Si la shard key tiene baja cardinalidad o distribucion desigual, se generan hotspots: chunks con muchos datos o peticiones concentrados en un solo shard.
- Una shard key monoticamente creciente (como un timestamp) concentra todas las escrituras en el ultimo shard, porque los valores nuevos caen siempre en el chunk del extremo superior.

### Hashed sharding

MongoDB calcula un hash MD4 del valor de la shard key y utiliza el resultado para determinar la distribucion. Los documentos se distribuyen segun el hash, no segun el valor original.

```javascript
sh.shardCollection("midb.usuarios", { email: "hashed" })
```

Ventajas:
- Distribucion practicamente uniforme de los datos, independientemente de la cardinalidad o distribucion de la shard key.
- Elimina el problema de las shard keys monotonas: el hash de un timestamp se distribuye aleatoriamente entre todos los shards.

Desventajas:
- Las consultas por rango sobre la shard key no pueden ser dirigidas a un shard especifico. mongos debe enviar la consulta a todos los shards (scatter-gather).
- Solo soporta igualdad en las consultas con shard key.

> La regla general es: si las consultas principales son por igualdad (buscar un documento por su ID o email), usar hashed. Si las consultas principales son por rango (buscar documentos entre dos fechas), usar ranged.

