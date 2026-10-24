# WiredTiger: Motor de almacenamiento

WiredTiger es el motor de almacenamiento de MongoDB desde la version 4.2, cuando se elimino definitivamente el soporte para MMAPv1. Su diseno responde a la necesidad de ofrecer alto rendimiento en cargas de trabajo intensivas en escritura, uso eficiente del espacio en disco y comportamiento predecible bajo contencion. Comprender su funcionamiento interno resulta esencial para dimensionar correctamente un servidor MongoDB y diagnosticar problemas de rendimiento.

## Compresion de datos

WiredTiger comprime los datos tanto en disco como en las paginas que intercambia con el sistema de archivos. La compresion opera a nivel de pagina (por defecto de 4 KB a 32 KB) y afecta tanto a los datos de las colecciones como a los indices.

MongoDB expone tres algoritmos de compresion configurables a traves del parametro `storage.wiredTiger.collectionConfig.blockCompressor`:

- **snappy** (predeterminado): optimizado para velocidad. Ofrece una tasa de compresion modesta (tipicamente 2:1 a 3:1) pero con una sobrecarga de CPU minima. Es la opcion recomendada para la mayoria de las cargas de trabajo, ya que el ahorro en E/S de disco compensa ampliamente el coste de compresion.
- **zlib**: proporciona una compresion significativamente mayor (3:1 a 5:1) a costa de un uso de CPU mas elevado. Adecuado cuando el espacio en disco es limitado o cuando el almacenamiento es mas costoso que el procesamiento.
- **zstd**: equilibrio entre snappy y zlib. Se aproxima a la relacion de compresion de zlib con una velocidad cercana a snappy. Introducido como opcion en MongoDB 4.2.

La eleccion del compresor depende del perfil de la carga de trabajo. En sistemas con CPU limitada, snappy es la opcion segura. En almacenes de datos historicos o backups, zstd ofrece la mejor relacion compresion/velocidad.

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd
```

La configuracion se aplica por coleccion y debe establecerse en el momento de creacion de la coleccion; no es posible cambiarla a posteriori sin recrear la coleccion.

## Cache de WiredTiger

WiredTiger mantiene un cache en memoria que almacena las paginas de datos e indices mas utilizadas. El tamano de este cache es probablemente el parametro de configuracion mas influyente en el rendimiento de MongoDB.

Por defecto, WiredTiger asigna el 50 % de la RAM disponible (menos 1 GB) al cache interno. En sistemas con 8 GB de RAM, el cache de WiredTiger tomara aproximadamente 3.5 GB. Este valor puede ajustarse mediante:

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
```

Cuando el cache se llena, WiredTiger debe desalojar paginas para dejar espacio a las nuevas. El algoritmo de desalojo no es un LRU clasico: utiliza una heuristica que combina la frecuencia de acceso, la antiguedad de la pagina y el coste de volver a leerla desde disco. Las paginas sucias (modificadas pero no escritas en disco) deben escribirse antes de ser desalojadas, lo que introduce una sobrecarga adicional.

El rendimiento del cache se monitoriza con la metricas `wiredTiger.cache` devueltas por `db.serverStatus()`:

```
wiredTiger.cache."bytes currently in the cache"
wiredTiger.cache."tracked dirty bytes in the cache"
wiredTiger.cache."unmodified pages evicted"
wiredTiger.cache."modified pages evicted"
```

Un numero alto de paginas sucias desalojadas indica que el cache es pequeno para la carga de escritura, lo que se traduce en mayor latencia de escritura. En ese caso, incrementar el tamano del cache puede aliviar la presion, siempre que el sistema disponga de RAM suficiente.

## Concurrencia y control de acceso

WiredTiger implementa control de concurrencia multiversion (MVCC) optimista a nivel de documento. Esto significa que cada operacion de lectura opera sobre una snapshot consistente de los datos sin necesidad de adquirir bloqueos. Las escrituras, por su parte, adquieren bloqueos exclusivos a nivel de documento, pero no bloquean las lecturas de otros documentos.

El modelo de concurrencia de WiredTiger se basa en transacciones. Cada operacion en MongoDB (incluso una simple `find()`) se ejecuta dentro de una transaccion implicita. Cuando una escritura intenta modificar un documento que otra transaccion ha modificado pero aun no ha confirmado, WiredTiger detecta el conflicto y la transaccion entrante reintenta la operacion (write conflict). Si el conflicto persiste, la operacion falla con un error y el controlador del cliente debe reintentarla.

Este enfoque optimista funciona bien cuando la contencion sobre documentos individuales es baja, que es el caso comun en MongoDB. Sin embargo, en escenarios donde multiples hilos compiten por el mismo documento (por ejemplo, un contador global), los write conflicts pueden degradar el rendimiento. En esos casos, el patron de diseno debe reconsiderarse (por ejemplo, usando operaciones atomicas como `$inc` en lugar de leer-actualizar-escribir).

WiredTiger soporta multiples niveles de aislamiento:

- **read-uncommitted**: permite leer datos no confirmados.
- **read-committed**: lee solo datos confirmados (nivel por defecto).
- **snapshot**: cada transaccion ve una snapshot consistente del inicio.

MongoDB configura WiredTiger para usar snapshot isolation en las lecturas y read-committed en las escrituras, lo que proporciona un equilibrio optimo entre consistencia y rendimiento.

## Journaling y durabilidad

El journal de WiredTiger garantiza que, ante un fallo del sistema (corte de energia, caida del proceso), no se pierdan las escrituras confirmadas. Funciona como un registro de escritura anticipada (write-ahead log, WAL) donde cada modificacion se persiste antes de aplicarse a los archivos de datos principales.

El flujo de una escritura con journaling habilitado es:

1. La aplicacion emite una escritura (por ejemplo, `insertOne`).
2. `mongod` recibe la operacion y la delega en WiredTiger.
3. WiredTiger aplica el cambio en memoria (cache) y registra la operacion en el journal.
4. Transcurrido el intervalo de confirmacion (por defecto 100 ms) o si se supera el umbral de datos en el journal (por defecto 100 MB), WiredTiger sincroniza el journal en disco (`fsync`).
5. Periodicamente, WiredTiger escribe las paginas sucias de la cache en los archivos de datos (checkpoint).

El intervalo de confirmacion del journal se configura con:

```yaml
storage:
  journal:
    commitIntervalMs: 50
```

Reducir este intervalo mejora la durabilidad (menos datos en riesgo ante un fallo) pero incrementa la frecuencia de operaciones de E/S sincronas, lo que puede afectar al rendimiento. El valor por defecto de 100 ms representa un compromiso razonable.

El write concern de MongoDB permite al cliente decidir el nivel de durabilidad que requiere. Con `{ w: 1, j: true }`, el cliente espera a que el journal se haya escrito en disco antes de recibir la confirmacion. Con `{ w: 1, j: false }`, la escritura se confirma en cuanto se aplica en memoria, sin esperar al journal, lo que es mas rapido pero expone a perdida de datos si el servidor falla antes del siguiente commit del journal.

> El journal no es un sustituto de los backups. Sirve para recuperar el estado del sistema tras un fallo, pero no protege contra corruption de datos, errores logicos o eliminaciones accidentales. Para esos casos se requieren backups periodicos y, preferiblemente, un replica set con un miembro retrasado.

## Checkpoints

WiredTiger escribe periodicamente un checkpoint, que es una snapshot consistente de los datos en un instante determinado. Durante un checkpoint, todas las paginas sucias de la cache se sincronizan en los archivos de datos. Una vez completado, el journal puede truncarse hasta ese punto.

Los checkpoints se producen cada 60 segundos por defecto, aunque el intervalo puede ajustarse. Si el servidor se recupera tras un fallo, WiredTiger replaya las entradas del journal desde el ultimo checkpoint hasta el momento del fallo, restaurando el estado exacto de los datos en ese instante.

La combinacion de checkpoints periodicos y journaling continuo permite a WiredTiger ofrecer durabilidad sin necesidad de sincronizar cada escritura individual en disco, lo que reduce drasticamente la E/S total del sistema.

