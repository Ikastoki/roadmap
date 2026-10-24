# Read Preferences, Write Concerns y Read Concerns

La replicacion en MongoDB introduce un espacio de configuracion que permite equilibrar consistencia, disponibilidad y rendimiento segun las necesidades de cada operacion. No todas las lecturas requieren ver la escritura mas reciente, ni todas las escrituras necesitan ser confirmadas por todos los nodos antes de responder al cliente. Los read preferences, write concerns y read concerns son los mecanismos que ofrece MongoDB para controlar este equilibrio.

## Read Preferences

La read preference determina que nodo del replica set atiende una operacion de lectura. Por defecto, MongoDB envia todas las lecturas al primario (modo `primary`), lo que garantiza consistencia inmediata pero concentra toda la carga en un solo nodo.

```javascript
// Configurar read preference en el driver (Node.js)
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred'
});
```

### primary (default)

Todas las lecturas se dirigen al primario. Garantiza consistencia monotonicamente creciente (cada lectura ve al menos los datos de la lectura anterior). Es la opcion mas segura pero la que menos escala.

### primaryPreferred

Lee del primario si esta disponible; si no, lee de un secundario. Util durante failovers para evitar errores de lectura cuando el primario esta temporalmente inaccesible.

### secondary

Todas las lecturas se dirigen exclusivamente a secundarios. No se lee del primario bajo ninguna circunstancia. Escala la carga de lectura pero los datos pueden estar desactualizados respecto al primario.

### secondaryPreferred

Lee de secundarios si hay alguno disponible; si no, cae al primario. Es la configuracion mas comun para equilibrar carga de lectura sin arriesgar disponibilidad.

### nearest

Lee del nodo con menor latency de red, independientemente de si es primario o secundario. Util en despliegues multi-region para reducir la latencia de las lecturas.

```javascript
// nearest tiene en cuenta la latencia de red
const client = new MongoClient(uri, {
  readPreference: 'nearest',
  localThresholdMS: 15 // Solo nodos dentro de 15ms del mas cercano
});
```

> Las read preferences se pueden definir a nivel de cliente, base de datos, coleccion o incluso por operacion individual con metodos como `cursor.readPref()`.

## Write Concerns

El write concern define cuantos nodos deben confirmar una escritura antes de que MongoDB notifique exito al cliente. Controla el equilibrio entre seguridad de los datos y latencia de escritura.

```javascript
// Write concern a nivel de operacion
db.collection.insertOne(
  { nombre: "ejemplo" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
);
```

### w

El parametro `w` especifica cuantos nodos deben confirmar la escritura:

- **w: 1** (default): solo el primario confirma. Es el mas rapido. Si el primario falla antes de replicar, la escritura se pierde.
- **w: "majority"**: la mayoria de los miembros votantes confirman. Garantiza que la escritura sobrevive a un failover.
- **w: 0**: no espera confirmacion. Util para metricas o logs donde la perdida de datos es aceptable. No se capturan errores de red ni de disco.
- **w: <numero>**: espera confirmacion de N nodos. Si el replica set tiene 5 miembros, `w: 4` espera confirmacion de 4 nodos.

```javascript
// Ver el tag "majority" para el replica set actual
rs.conf().getLastErrorDefaults;
```

### j

El parametro `j` (journal) especifica si la escritura debe ser commiteada al journal antes de confirmar:

- **j: true**: espera a que la escritura se haya escrito en el journal de cada nodo. Protege contra cortes de energia.
- **j: false**: no espera journal commit. La escritura queda solo en la cache de WiredTiger.

### wtimeout

Tiempo maximo en milisegundos que MongoDB espera a que se cumpla el write concern. Si el write concern no se satisface dentro del timeout, la operacion retorna un error pero la escritura no se deshace (puede haberse aplicado en algunos nodos y en otros no).

```javascript
try {
  db.collection.insertOne(
    { data: "test" },
    { writeConcern: { w: "majority", wtimeout: 100 } }
  );
} catch (e) {
  print("Write concern timeout: " + e);
}
```

## Read Concerns

Mientras que el read preference controla *donde* se lee, el read concern controla *que* datos se consideran visibles. Permite decidir si una lectura debe esperar a que los datos hayan sido replicados a la mayoria de nodos o si puede devolver datos que aun no se han propagado completamente.

```javascript
db.collection.find({}).readConcern("majority");
```

### local (default)

Devuelve los datos mas recientes disponibles en el nodo al que se consulta. Si se lee de un secundario que aun no ha replicado una escritura reciente, la lectura no la reflejara. Es el read concern mas rapido.

### majority

Devuelve solo datos que han sido confirmados por la mayoria de los miembros del replica set. Garantiza que los datos leidos no se revertiran en caso de un failover. Para usar `majority` es necesario habilitar el almacenamiento de la mayoria en el replica set.

```javascript
// Habilitar majority read concern en la configuracion
rs.conf().members.forEach(m => {
  m.settings = { ...m.settings, majoritReadConcern: true };
});
```

### linearizable

Ofrece la garantia mas fuerte de consistencia: la lectura devuelve el valor mas reciente que ha sido commiteado por la mayoria, y confirma que ningun otro nodo ha escrito un valor mas reciente antes de responder. Requiere consultar a un solo nodo (no se puede combinar con read preferences que envien a varios nodos). Tiene latencia alta porque implica una ronda de confirmacion entre los nodos.

```javascript
db.collection.find({ _id: 1 }).readConcern("linearizable");
```

> Linearizable read concern solo esta disponible para operaciones que especifiquen un filtro unico por `_id`.

### available (para sharded clusters)

Disponible solo en clusters sharded, devuelve los datos mas recientes del shard consultado sin verificar que sean consistentes con otros shards. No garantiza que todos los shards hayan visto el mismo conjunto de escrituras.

## Failover

El failover es el proceso por el cual un replica set recupera la capacidad de aceptar escrituras tras la caida del primario. Es automatico y no requiere intervencion administrativa en condiciones normales.

1. Los secundarios detectan que el primario no responde heartbeats (10 segundos por defecto).
2. Se inicia una eleccion: los secundarios elegibles votan por un nuevo primario.
3. El ganador asume el rol de primario y comienza a aceptar escrituras.
4. El resto de nodos se convierten en secundarios del nuevo primario.
5. Cuando el nodo caido se recupera, se reincorpora como secundario.

El tiempo total del failover suele estar entre 10 y 30 segundos. Durante ese periodo el replica set no acepta escrituras. Las lecturas pueden continuar si la read preference lo permite.

```javascript
// Forzar stepdown manual para mantenimiento
rs.stepDown(120, 30); // 120s sin aceptar primario, 30s de gracia
```

La aplicacion debe estar preparada para manejar errores de conexion durante el failover. Los drivers oficiales de MongoDB incorporan mecanismos de reconexion automatica y reintento de operaciones (retryable writes y retryable reads).

> El parametro `electionTimeoutMillis` en la configuracion del replica set controla cuanto tiempo esperan los secundarios antes de declarar al primario como caido. Valores mas bajos aceleran el failover pero aumentan el riesgo de falsos positivos por latencia de red.

