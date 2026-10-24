# Transacciones en MongoDB

Hasta la version 4.0, MongoDB ofrecia garantias atomicas unicamente a nivel de un solo documento: cualquier operacion que modificara un unico documento se ejecutaba de forma atomica, pero no existia un mecanismo nativo para coordinar cambios en multiples documentos (o multiples colecciones) con garantias ACID. Con la introduccion de las transacciones multi-documento en MongoDB 4.0 (y su extension a sharded clusters en 4.2), el motor de bases de datos dio un salto cualitativo que lo situo a la par de los sistemas relacionales en terminos de garantias transaccionales, manteniendo al mismo tiempo el modelo documental y la capacidad de escalar horizontalmente. Una transaccion multi-documento permite agrupar varias operaciones de lectura y escritura en una unidad logica que se ejecuta completamente o se deshace por completo, preservando la consistencia de los datos incluso en entornos distribuidos.

## ACID en MongoDB 4.0+

Las transacciones multi-documento en MongoDB implementan las cuatro propiedades ACID, aunque con matices propios del modelo documental y la arquitectura distribuida subyacente.

### Atomicidad

Todas las operaciones incluidas en una transaccion se aplican como una unica unidad indivisible. Si cualquier operacion dentro de la transaccion falla, todas las modificaciones realizadas hasta ese punto se deshacen automaticamente. Esto es posible gracias al _write-ahead log_ (WiredTiger) que registra los cambios pendientes antes de aplicarlos a los archivos de datos. Si el servidor se cae antes de completar el commit, el log permite restaurar el estado previo.

### Consistencia

Las transacciones respetan las reglas de integridad definidas en la base de datos: indices unicos, tipos de datos, validaciones de esquema y cualquier otra restriccion. Si una operacion dentro de la transaccion viola alguna restriccion, la transaccion es abortada en su totalidad. Ningun estado intermedio queda visible para otras operaciones.

### Aislamiento

MongoDB implementa aislamiento mediante _snapshots_ (multiversion concurrency control, MVCC). Cada transaccion opera sobre una instantanea consistente de los datos tomada al iniciarse. Los cambios realizados por otras transacciones concurrentes no son visibles hasta que la transaccion actual hace commit. El nivel de aislamiento equivalente es _snapshot isolation_, que evita lecturas sucias, lecturas no repetibles y lecturas fantasma.

### Durabilidad

Una vez que `commitTransaction` retorna con exito, los cambios estan persistedos en el journal de WiredTiger y replicados al numero de nodos configurado mediante `writeConcern`. Si el servidor primario falla inmediatamente despues del commit, los datos pueden recuperarse desde el journal o desde los nodos secundarios que hayan recibido la replicacion.

> A diferencia de las bases de datos relacionales tradicionales, MongoDB no utiliza bloqueos a nivel de fila o tabla para garantizar el aislamiento. El uso de MVCC basado en snapshots permite un alto grado de concurrencia sin los cuellos de botella tipicos de los mecanismos de bloqueo clasicos.

## startTransaction

El ciclo de vida de una transaccion comienza con la creacion de una sesion y la llamada a `startTransaction`. En la mayoria de los controladores modernos, se utiliza el metodo `startSession()` del objeto cliente para obtener una sesion, y sobre ella se invoca `startTransaction()`.

```javascript
const session = client.startSession();
session.startTransaction({
  readConcern: { level: 'snapshot' },
  writeConcern: { w: 'majority' }
});
```

El objeto `session` identifica de forma unica la transaccion y debe asociarse a todas las operaciones que forman parte de ella, pasandolo como parametro `session` en cada operacion. Sin ese vinculo explicito, las operaciones se ejecutarian fuera de la transaccion.

## commitTransaction

Confirma todas las operaciones de la transaccion y hace persistentes los cambios. MongoDB ejecuta un protocolo interno de commit en dos fases cuando la transaccion involucra multiples fragmentos en un cluster shardeado. Si el commit tiene exito, todos los cambios son visibles para operaciones posteriores.

```javascript
await session.commitTransaction();
```

Si ocurre un error durante el commit (por ejemplo, un conflicto de escritura o un fallo de red), el controlador lanza una excepcion que debe manejarse explicitamente. Es en ese punto donde tiene sentido reintentar la transaccion completa.

## abortTransaction

Descarta todos los cambios realizados dentro de la transaccion y libera los recursos asociados (snapshots, locks de WiredTiger, buffers de escritura). Puede invocarse explicitamente o de forma implicita cuando se detecta un error antes del commit.

```javascript
await session.abortTransaction();
```

Una vez abortada o commiteada, la sesion puede reutilizarse para una nueva transaccion o cerrarse con `session.endSession()`.

## withTransaction

Para evitar el codigo boilerplate de gestion manual de reintentos, MongoDB proporciona el metodo `withTransaction` sobre el objeto de sesion. Este metodo envuelve la logica de la transaccion en un callback que se ejecuta dentro de la transaccion y, ademas, reintenta automaticamente la transaccion completa en caso de errores transitorios (como `TransientTransactionError` o `UnknownTransactionCommitResult`).

```javascript
const session = client.startSession();

try {
  await session.withTransaction(async () => {
    const coll1 = db.collection('cuentas');
    const coll2 = db.collection('movimientos');

    await coll1.updateOne(
      { _id: origenId },
      { $inc: { saldo: -100 } },
      { session }
    );
    await coll1.updateOne(
      { _id: destinoId },
      { $inc: { saldo: 100 } },
      { session }
    );
    await coll2.insertOne(
      { tipo: 'transferencia', origen: origenId, destino: destinoId, monto: 100 },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

`withTransaction` maneja internamente:
- Reintento si falla `commitTransaction` con `UnknownTransactionCommitResult`.
- Reintento si la transaccion falla con `TransientTransactionError` (tipicamente por conflictos de escritura entre transacciones concurrentes).
- La invocacion de `abortTransaction` si el callback lanza una excepcion no recuperable.

> El numero maximo de reintentos en `withTransaction` no es configurable. El metodo reintenta hasta agotar un tiempo maximo definido por `maxCommitTimeMS` (por defecto 10 segundos) o hasta que el error no sea recuperable.

## Retryable writes

Las escrituras reintentables son un mecanismo independiente de las transacciones, pero complementario para construir sistemas resilientes. Cuando `retryWrites=true` en la cadena de conexion, el controlador reintenta automaticamente operaciones de escritura individuales (insertOne, updateOne, deleteOne, findOneAndUpdate, etc.) si se produce un error de red o si el nodo primario cambia durante la operacion.

```
mongodb://localhost:27017/mibd?retryWrites=true
```

Las transacciones tambien se benefician de este mecanismo: si el commit falla por un cambio de primario, el controlador puede reintentar la operacion de commit contra el nuevo primario. Sin embargo, `retryWrites` no reintenta la transaccion entera si falla una operacion intermedia; eso es responsabilidad de `withTransaction` o del codigo de la aplicacion.

## Read concern majority y Write concern majority

El nivel de garantia de lectura y escritura se configura por transaccion y determina como se comporta MongoDB ante fallos y replicacion.

### readConcern majority

Garantiza que los datos leidos dentro de la transaccion han sido confirmados por la mayoria de los nodos del conjunto de replicas. Esto asegura que no se leen datos que puedan ser revertidos posteriormente por un rollback. En transacciones se recomienda usar `readConcern: 'majority'` o `readConcern: 'snapshot'` (que es una variante mas estricta que garantiza una instantanea completa y consistente).

```javascript
session.startTransaction({
  readConcern: { level: 'snapshot' }
});
```

### writeConcern majority

Garantiza que las escrituras de la transaccion se consideran commiteadas solo cuando la mayoria de los nodos del conjunto de replicas las han aplicado. Sin `w: 'majority'`, una escritura podria confirmarse en el primario pero perderse si el primario cae antes de replicar.

```javascript
session.startTransaction({
  writeConcern: { w: 'majority' }
});
```

> El valor por defecto de `writeConcern` en MongoDB es `{ w: 1 }` (confirmacion del primario unicamente). Para transacciones que requieren durabilidad absoluta, debe establecerse explicitamente `{ w: 'majority' }`.

## Opciones de transaccion

La llamada a `startTransaction` acepta un objeto de opciones que permite personalizar el comportamiento de la transaccion.

```javascript
session.startTransaction({
  readConcern: { level: 'snapshot' },
  writeConcern: { w: 'majority', j: true, wtimeout: 5000 },
  readPreference: 'primary',
  maxTimeMS: 30000
});
```

### readConcern

Define la garantia de consistencia de las lecturas dentro de la transaccion. Los valores posibles son `'local'`, `'majority'` y `'snapshot'`. Para transacciones que requieren aislamiento total, `'snapshot'` es la opcion recomendada porque asegura que todas las lecturas ven una instantanea consistente de los datos.

### writeConcern

Define la garantia de persistencia de las escrituras. Ademas de `w`, pueden configurarse:
- `j: true`: exige que la escritura se haya persistido en el journal antes de confirmar.
- `wtimeout`: tiempo maximo (en milisegundos) para esperar la confirmacion del write concern. Si se supera, la operacion falla con un error.

### readPreference

Define hacia que nodo del conjunto de replicas se dirigen las lecturas de la transaccion. Para transacciones, el unico valor seguro es `'primary'`, dado que las lecturas en secundarios podrian devolver datos desactualizados. MongoDB impone esta restriccion: las operaciones de lectura en una transaccion deben ejecutarse contra el primario.

### maxTimeMS

Tiempo maximo en milisegundos que la transaccion puede permanecer abierta. Pasado ese limite, MongoDB aborta automaticamente la transaccion. El valor por defecto es el definido por `transactionLifetimeLimitSeconds` en el servidor (60 segundos).

```javascript
session.startTransaction({
  maxTimeMS: 30000
});
```

> Aunque se configure `maxTimeMS` en la transaccion, el limite global del servidor (`transactionLifetimeLimitSeconds`, por defecto 60s) siempre prevalece. Una transaccion que exceda ese limite sera abortada por el servidor independientemente del valor local.

