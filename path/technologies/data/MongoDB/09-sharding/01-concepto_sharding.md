# Sharding: particion horizontal en MongoDB

El crecimiento de los datos es uno de los problemas mas complejos de resolver en cualquier sistema de bases de datos. Cuando una coleccion alcanza varios cientos de gigabytes o el rendimiento de escritura se satura porque un solo nodo no da abasto, las soluciones convencionales (indices, optimizacion de consultas, escalado vertical) dejan de ser efectivas. MongoDB resuelve este limite mediante sharding, una tecnica de particion horizontal que distribuye los datos entre multiples servidores independientes, permitiendo que el sistema escale de forma practicamente ilimitada.

## Concepto de particion horizontal

El sharding divide una coleccion en fragmentos mas pequenos llamados chunks y los distribuye entre varios servidores (shards). Cada shard es un replica set independiente que contiene un subconjunto de los datos. Ningun shard tiene los datos completos, pero todos juntos forman la coleccion completa desde la perspectiva del cliente.

A diferencia del escalado vertical (anyadir mas CPU, RAM o disco al mismo servidor), el escalado horizontal permite anyadir maquinas enteras al cluster. Esto no solo incrementa la capacidad de almacenamiento, sino que tambien distribuye la carga de lecturas y escrituras entre multiples nodos.

> El sharding es transparente para la aplicacion: el cliente se conecta al router (mongos) y este se encarga de dirigir cada operacion al shard correspondiente sin que la aplicacion tenga que conocer la topologia del cluster.

## Componentes del sharded cluster

Un cluster shardeado en MongoDB se compone de tres elementos fundamentales que trabajan de forma coordinada:

### mongos (router de consultas)

El proceso mongos actua como puerta de entrada al cluster. No almacena datos persistentes; su unica funcion es interpretar las consultas de la aplicacion, consultar los metadatos en los config servers para determinar que shard posee los datos solicitados y reenviar la operacion al shard correcto. La aplicacion nunca se conecta directamente a los shards, sino siempre a traves de mongos.

En un cluster de produccion es recomendable ejecutar varios procesos mongos (tipicamente uno por aplicacion o instancia de despliegue) para evitar que se convierta en un punto unico de fallo.

### Config servers (servidores de configuracion)

Los config servers almacenan los metadatos del cluster: que shards existen, que colecciones estan shardeadas, como se distribuyen los chunks entre los shards y el rango de valores que cubre cada chunk. Estos metadatos son esenciales para que mongos pueda enrutar correctamente las consultas.

Desde MongoDB 3.4, los config servers deben ejecutarse como un replica set (CSRS - Config Server Replica Set). Un CSRS tipico tiene 3 miembros, lo que garantiza alta disponibilidad de los metadatos. Si los config servers fallan, el cluster se vuelve inoperativo porque mongos no puede determinar donde estan los datos.

### Shards (nodos de datos)

Cada shard es un replica set independiente que contiene una parte de los datos del cluster. Los shards pueden estar compuestos por uno o varios nodos, pero en produccion cada shard debe ser un replica set de al menos 3 miembros para garantizar tolerancia a fallos.

Cuando se anyade un nuevo shard al cluster, el balancer comienza a migrar chunks desde los shards existentes hacia el nuevo, redistribuyendo la carga de forma automatica.

## La shard key como eje del sharding

La shard key es el campo (o combinacion de campos) que MongoDB utiliza para decidir en que shard debe almacenarse cada documento. Es, sin exagerar, la decision de diseno mas importante que se toma al configurar un cluster shardeado.

Cuando un documento se inserta, MongoDB evalua el valor de la shard key, lo asigna a un rango de chunks y lo envia al shard responsable de ese rango. De la misma forma, cuando se realiza una consulta que incluye la shard key, mongos puede dirigir la operacion unicamente al shard o shards que contienen los datos relevantes (consulta dirigida). Si la consulta no incluye la shard key, mongos debe enviarla a todos los shards (scatter-gather), lo que es significativamente menos eficiente.

> La shard key no puede modificarse una vez que la coleccion ha sido shardeada. Elegir una shard key inadecuada puede degradar el rendimiento de forma irreversible, obligando a migrar los datos a una nueva coleccion con una shard key diferente.

Una shard key efectiva debe cumplir tres propiedades fundamentales:

- **Alta cardinalidad**: cuantos mas valores distintos tenga la shard key, mas granular sera la distribucion de chunks y mejor se repartira la carga entre los shards.
- **Baja frecuencia de cambio**: los valores de la shard key no deberian cambiar o, si lo hacen, debe ser con poca frecuencia. Cambiar la shard key de un documento puede obligar a MongoDB a migrar el chunk entero a otro shard.
- **Distribucion uniforme**: los valores de la shard key deben distribuirse de forma homogenea para evitar que un shard reciba significativamente mas carga que los demas (hotspot).

