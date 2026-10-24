# Configuracion de un Replica Set

Montar un replica set no requiere herramientas externas ni configuracion compleja: basta con iniciar varias instancias de mongod con el mismo nombre de replica set y ejecutar unos pocos comandos en la shell de MongoDB. El propio sistema se encarga de la sincronizacion inicial, la deteccion de miembros y el mantenimiento del estado. La configuracion se almacena en el documento `local.system.replset` y se replica a todos los miembros.

## Inicializacion con rs.initiate

El primer paso es arrancar las instancias mongod con el parametro `replSet`:

```bash
mongod --replSet "miReplicaSet" --port 27017 --dbpath /data/rs1
mongod --replSet "miReplicaSet" --port 27018 --dbpath /data/rs2
mongod --replSet "miReplicaSet" --port 27019 --dbpath /data/rs3
```

Una vez arrancadas, se conecta a una de ellas (normalmente la que sera primaria) y se ejecuta `rs.initiate()`:

```javascript
rs.initiate({
  _id: "miReplicaSet",
  members: [
    { _id: 0, host: "localhost:27017" },
    { _id: 1, host: "localhost:27018" },
    { _id: 2, host: "localhost:27019" }
  ]
});
```

El metodo `rs.initiate()` crea la configuracion inicial del replica set. El parametro `_id` debe coincidir con el valor de `replSet` usado al arrancar los mongod. Cada miembro necesita un `_id` unico dentro del replica set (no confundir con el ObjectId de los documentos) y su direccion de red completa.

> `rs.initiate()` es idempotente: si el replica set ya existe, retorna un error. Para modificar la configuracion se usa `rs.reconfig()`.

## Agregar y eliminar miembros

Una vez el replica set esta activo, se pueden anadir o eliminar nodos sin detener el sistema.

```javascript
// Agregar un nuevo miembro
rs.add("localhost:27020");

// Agregar con configuracion avanzada
rs.add({ host: "localhost:27020", priority: 0, hidden: true });

// Agregar un arbiter
rs.addArbiter("localhost:27021");

// Eliminar un miembro
rs.remove("localhost:27020");
```

Cuando se agrega un nuevo miembro, este realiza una sincronizacion inicial (initial sync) copiando todos los datos de otro nodo del replica set. Dependiendo del volumen de datos, este proceso puede tomar minutos u horas. Durante la sincronizacion inicial el nodo aparece en estado `STARTUP` o `RECOVERING`.

## rs.status

`rs.status()` es el comando diagnostico mas utilizado. Proporciona una instantanea del estado actual del replica set desde la perspectiva del nodo al que se consulta:

```javascript
rs.status();
```

La salida incluye:
- `set`: nombre del replica set.
- `myState`: estado del nodo actual (0=STARTUP, 1=PRIMARY, 2=SECONDARY, 3=RECOVERING, 7=ARBITER).
- `members`: array con informacion detallada de cada miembro: estado, ultimo heartbeat, lag de replicacion, uptime.
- `optimes`: marcas de tiempo de la ultima operacion commiteada en cada nodo.
- `electionParticipantMetrics`: metricas de la ultima eleccion.

```javascript
rs.status().members.forEach(m => {
  print(m.name + ': ' + m.stateStr + ' (lag: ' + m.optimeDate + ')');
});
```

## rs.conf

`rs.conf()` devuelve el objeto de configuracion del replica set. A diferencia de `rs.status()`, que muestra el estado actual, `rs.conf()` muestra la configuracion activa definida por el administrador:

```javascript
rs.conf();
```

El objeto retornado contiene campos como `_id`, `version` (numero de configuracion, se incrementa con cada cambio), `members` (array de objetos con configuracion de cada miembro) y `settings` (opciones globales del replica set como `electionTimeoutMillis` o `heartbeatTimeoutSecs`).

```javascript
rs.conf().members.forEach(m => {
  print(m.host + ' priority: ' + m.priority + ' votes: ' + m.votes);
});
```

## Priority

La prioridad determina la probabilidad de que un miembro se convierta en primario durante una eleccion. Por defecto todos los miembros tienen prioridad 1. El miembro con la prioridad mas alta entre los candidatos ganara la eleccion.

```javascript
rs.add({ host: "localhost:27017", priority: 2 });
rs.add({ host: "localhost:27018", priority: 1 });
rs.add({ host: "localhost:27019", priority: 0 }); // Nunca sera primario
```

Un miembro con prioridad 0 nunca puede convertirse en primario ni desencadenar una eleccion. Es util para nodos que solo actuan como secundarios de solo lectura o para nodos en centros de datos secundarios que no deben asumir el liderazgo.

> Si el primario actual tiene prioridad mas baja que otro miembro, se produce un "reelection": el primario se retira voluntariamente para permitir que el nodo de mayor prioridad gane la eleccion.

## Votes

Por defecto cada miembro del replica set tiene 1 voto. Los miembros con votes=0 no participan en las elecciones. Esto es util cuando se superan los 7 miembros votantes maximos permitidos por replica set:

```javascript
rs.add({ host: "localhost:27017", votes: 0 });
```

> El numero maximo de miembros votantes en un replica set es 7. Si se necesitan mas nodos, se agregan con votes=0 para que no cuenten para el quorum.

## Hidden Members

Los miembros ocultos no aparecen en la salida de `rs.status()` para los clientes estandar (aunque si en la shell si se consulta directamente). No reciben trafico de lectura incluso si la read preference favorece secundarios. Su proposito principal es ejecutar tareas que no deben interferir con la carga de produccion: backups, analiticas o reportes.

```javascript
rs.add({ host: "localhost:27020", hidden: true, priority: 0 });
```

Un miembro oculto debe tener prioridad 0, ya que no tendria sentido que un nodo invisible para los clientes se convierta en primario.

## Delayed Members

Los miembros retardados mantienen una copia de los datos con un retraso intencionado respecto al primario. Sirven como seguro contra errores humanos: si alguien elimina datos accidentalmente, el nodo retardado aun conserva la informacion anterior a la operacion destructiva.

```javascript
rs.add({
  host: "localhost:27021",
  hidden: true,
  priority: 0,
  slaveDelay: 3600 // 1 hora de retraso
});
```

Es fundamental que un miembro retardado sea tambien oculto y tenga prioridad 0, pues nunca debe convertirse en primario (sus datos estan desactualizados intencionadamente). El retraso se especifica en segundos y debe ser mayor que el tiempo maximo que se considera aceptable para recuperar datos.

> Los delayed members no son un sustituto de los backups. Son una capa adicional de proteccion, pero no protegen contra fallos catastroficos como la corrupcion de datos que se replica con retraso.

