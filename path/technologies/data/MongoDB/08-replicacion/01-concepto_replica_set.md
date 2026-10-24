# Concepto de Replica Set

Un replica set en MongoDB es un grupo de procesos mongod que mantienen el mismo conjunto de datos, proporcionando redundancia y alta disponibilidad. La idea fundamental es que si uno de los nodos falla, los otros pueden continuar sirviendo datos sin intervencion manual. Esto convierte a MongoDB en un sistema tolerante a fallos de forma nativa: no requiere proxies externos ni soluciones de clustering propietarias. La arquitectura de replica set esta disenada en torno al concepto de liderazgo: un nodo actua como primario y el resto como secundarios, con un mecanismo de eleccion automatica que garantiza que siempre haya un nodo capaz de aceptar escrituras.

## Primary

El nodo primario es el unico miembro del replica set que puede recibir operaciones de escritura. Todas las inserciones, actualizaciones y eliminaciones pasan obligatoriamente por el. MongoDB garantiza que solo existe un primario activo en un momento dado, lo que evita conflictos de escritura concurrente. El primario registra todos los cambios en su oplog (operation log), un conjunto de documentos que describen cada modificacion realizada. Los secundarios leen este oplog de forma continua para replicar los cambios de forma asincrona.

```javascript
rs.status().members.forEach(m => {
  if (m.stateStr === 'PRIMARY') print('Primary: ' + m.name);
});
```

> El primario no es elegido por sus capacidades tecnicas, sino mediante un proceso de votacion. Cualquier nodo con prioridad mayor que cero puede convertirse en primario si recibe suficientes votos.

## Secondary

Los nodos secundarios mantienen una copia exacta de los datos del primario aplicando continuamente los cambios registrados en el oplog. Por defecto no aceptan escrituras, pero si pueden atender lecturas si la aplicacion configura una preferencia de lectura que lo permita (read preference). Esta capacidad de servir lecturas desde secundarios permite escalar horizontalmente la carga de lectura sin aumentar la presion sobre el primario.

Un secundario puede convertirse en primario si el nodo actual falla y se desencadena una nueva eleccion. Para ello debe ser elegible: tener prioridad mayor que cero y estar al dia en su replicacion (o dentro del margen de perdida aceptable).

```javascript
rs.status().members.forEach(m => {
  if (m.stateStr === 'SECONDARY') print('Secondary: ' + m.name);
});
```

Los secundarios tambien realizan tareas internas como la replicacion de indices y la actualizacion de estadisticas. No son simplemente receptores pasivos; participan activamente en el mantenimiento del replica set.

## Arbiter

Un arbiter es un nodo especial que no almacena datos pero participa en las elecciones. Su unica funcion es proporcionar un voto para alcanzar la mayoria cuando el numero de nodos de datos es par o cuando no se dispone de recursos para mantener un tercer nodo de datos completo. Esto es util en despliegues con solo dos nodos de datos: sin un arbiter, si un nodo falla el otro no puede alcanzar mayoria y el replica set quedaria en modo solo lectura.

```javascript
rs.addArbiter("host:port");
```

> Los arbiters no deberian usarse como sustituto de un tercer nodo de datos en produccion. Introducen riesgo operativo porque si el arbiter falla y uno de los nodos de datos tambien, el replica set pierde la mayoria.

## Heartbeat

Cada miembro del replica set envia un heartbeat (latido) a los demas miembros cada dos segundos. Este mecanismo permite detectar fallos de red o de nodo de forma temprana. Si un nodo no recibe heartbeat de otro durante un periodo de 10 segundos (por defecto), el nodo silencioso se marca como inaccesible y se inicia el proceso de eleccion si era el primario.

```javascript
rs.status().members.forEach(m => {
  print(m.name + ' lastHeartbeat: ' + m.lastHeartbeat);
});
```

El heartbeat no solo verifica que el nodo esta vivo, sino que tambien transporta informacion sobre el estado del replica set: quien es el primario actual, la configuracion mas reciente y el progreso del oplog de cada miembro.

## Elections

Cuando el primario deja de comunicarse con los demas miembros del replica set (por caida del proceso, fallo de red o particion), los secundarios detectan la ausencia de heartbeat y desencadenan una eleccion. El proceso sigue estos pasos:

1. Un nodo secundario se autoproclama candidato a primario e incrementa su term (numero de eleccion).
2. Envia solicitudes de voto al resto de nodos.
3. Los nodos receptores verifican que el candidato esta al dia con el oplog (no tiene un retraso mayor que el configurado).
4. Si el candidato recibe votos de la mayoria de los miembros del replica set (incluyendose a si mismo), se convierte en primario.
5. El nuevo primario comienza a aceptar escrituras y los demas nodos se convierten en sus secundarios.

El tiempo tipico de una eleccion esta entre 10 y 30 segundos, durante los cuales el replica set no acepta escrituras. Las lecturas pueden seguir sirviendose si la aplicacion usa read preferences que permitan leer de secundarios.

```javascript
rs.stepDown(60); // Forzar eleccion para mantenimiento planificado
```

> Las elecciones siguen el algoritmo de consensus Raft adaptado. MongoDB no utiliza Paxos ni Raft directamente, sino un protocolo propio inspirado en Raft llamado "MongoDB Replica Set Protocol".

