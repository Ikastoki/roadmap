# Cuando usar transacciones

Las transacciones multi-documento son una herramienta potente, pero no deben convertirse en el mecanismo por defecto para toda operacion que involucre multiples documentos. MongoDB esta optimizado para operaciones atomicas a nivel de documento, y el diseno del esquema deberia, en la medida de lo posible, reflejar esa realidad. Las transacciones existen para cubrir los casos donde el modelo documental por si solo no puede garantizar la consistencia que la logica de negocio exige. Usarlas indiscriminadamente introduce complejidad, sobrecarga de rendimiento y riesgo de errores en tiempo de ejecucion que podrian evitarse con un mejor diseno del modelo de datos.

## Limitaciones

Conocer las restricciones tecnicas de las transacciones es fundamental para disenar sistemas robustos y evitar sorpresas en produccion.

### Colecciones no permitidas

No pueden utilizarse dentro de una transaccion las siguientes colecciones:
- Colecciones _capped_ (de tamano fijo).
- Colecciones en bases de datos de sistema (`admin`, `config`, `local`).
- Colecciones que ya contengan documentos que excedan el tamano maximo permitido.

### Operaciones DDL prohibidas

Dentro de una transaccion no se permiten operaciones de definicion de esquema (DDL):
- `createCollection`, `dropCollection`, `createIndexes`, `dropIndexes`.
- `collMod` (modificacion de opciones de coleccion).
- `renameCollection`.

Cualquier intento de ejecutar estas operaciones dentro de una transaccion lanza un error inmediato.

### Limite de 16 MB por operacion

Cada operacion individual dentro de una transaccion esta limitada por el tamano maximo de documento de MongoDB (16 MB). Este limite no se refiere al tamano total de la transaccion, sino a cada documento individual involucrado en una operacion de escritura. Si una actualizacion implica documentos que, sumados, exceden este limite, la operacion fallara.

### Duracion maxima de 60 segundos

`transactionLifetimeLimitSeconds` establece un limite global de 60 segundos para cualquier transaccion. Si una transaccion permanece abierta mas alla de este tiempo, el servidor la aborta automaticamente. Esto incluye el tiempo de espera entre operaciones, no solo el tiempo de ejecucion. Por ello, las transacciones deben ser breves: abrir la transaccion, ejecutar las operaciones necesarias y hacer commit lo antes posible.

### Otras restricciones importantes

- No se permiten operaciones de lectura en colecciones de la base de datos `config`.
- Las transacciones en un cluster shardeado no pueden involucrar mas de 1000 colecciones (limite del protocolo de commit en dos fases).
- Las lecturas dentro de una transaccion siempre se dirigen al nodo primario (`readPreference` fijado a `primary`).
- No es posible abrir varias transacciones simultaneas sobre una misma sesion; una sesion solo puede tener una transaccion activa a la vez.

> En un cluster shardeado, si una transaccion involucra documentos en diferentes fragmentos, el coordinador ejecuta un commit en dos fases. Si algun fragmento no responde dentro del tiempo limite, la transaccion se aborta. Disenar transacciones que trabajen sobre el menor numero posible de fragmentos reduce drasticamente la latencia y la probabilidad de fallo.

## Cuando usar transacciones multi-documento

Las transacciones estan justificadas en escenarios donde la integridad de los datos depende de que varias escrituras en documentos distintos (potencialmente en colecciones distintas) se ejecuten como una unidad atomica.

### Transferencias financieras

El caso clasico: debitar una cuenta y acreditar otra en una sola operacion atomica. Sin transacciones, un fallo entre el debito y el credito dejaria el sistema en un estado inconsistente (dinero que desaparece o que se crea de la nada).

```javascript
await session.withTransaction(async () => {
  await cuentas.updateOne(
    { _id: origen },
    { $inc: { saldo: -monto } },
    { session }
  );
  await cuentas.updateOne(
    { _id: destino },
    { $inc: { saldo: monto } },
    { session }
  );
});
```

### Pedidos con control de stock

Al registrar un pedido, es necesario verificar que hay stock disponible, reservarlo y crear el pedido. Si dos usuarios intentan comprar el ultimo articulo simultaneamente, la transaccion garantiza que solo uno lo consiga y el otro reciba un error.

```javascript
await session.withTransaction(async () => {
  const producto = await productos.findOne(
    { _id: productoId, stock: { $gte: cantidad } },
    { session }
  );
  if (!producto) throw new Error('Stock insuficiente');

  await productos.updateOne(
    { _id: productoId },
    { $inc: { stock: -cantidad } },
    { session }
  );
  await pedidos.insertOne(
    { usuarioId, productoId, cantidad, fecha: new Date() },
    { session }
  );
});
```

### Migraciones de datos con consistencia

Cuando se necesita transformar o migrar datos entre colecciones manteniendo la integridad referencial, una transaccion asegura que ningun lector vea el estado intermedio.

```javascript
await session.withTransaction(async () => {
  const docs = await coleccionVieja.find({}, { session }).toArray();
  await coleccionNueva.insertMany(docs.map(transformar), { session });
  await coleccionVieja.deleteMany({}, { session });
});
```

### Operaciones que requieren leer y escribir en el mismo ciclo

Cualquier operacion donde el valor a escribir depende del valor leido momentos antes necesita aislamiento. Sin transaccion, otra operacion concurrente podria modificar el documento entre la lectura y la escritura, provocando una condicion de carrera.

## Cuando NO usar transacciones

El uso excesivo de transacciones tiene un coste que a menudo no se justifica.

### Operaciones que afectan a un solo documento

Si la operacion modifica un unico documento, la atomicidad nativa de MongoDB es suficiente. No tiene sentido abrir una transaccion para, por ejemplo, actualizar un solo campo:

```javascript
// Esto NO necesita transaccion
await coleccion.updateOne({ _id: id }, { $set: { estado: 'activo' } });
```

### Lecturas sin escrituras

Las transacciones solo tienen sentido cuando hay al menos una operacion de escritura. Si solo se necesitan lecturas consistentes, `readConcern: 'majority'` o `readConcern: 'snapshot'` fuera de una transaccion proporcionan garantias suficientes sin la sobrecarga de gestionar una transaccion.

### Operaciones que pueden modelarse con documentos embebidos

Uno de los principios fundamentales de MongoDB es disenar el esquema para que las operaciones atomicas recaigan sobre un unico documento. Si los datos que necesitan actualizarse atomicamente pueden modelarse como un documento embebido, esa es casi siempre la mejor solucion.

**Mal diseno (requiere transaccion):**

```javascript
// usuarios: { _id, nombre }
// direcciones: { _id, usuarioId, calle, ciudad }
```

**Buen diseno (operacion atomica sin transaccion):**

```javascript
// usuarios: { _id, nombre, direccion: { calle, ciudad } }
```

## Impacto en rendimiento

Las transacciones multi-documento tienen un coste de rendimiento significativo en comparacion con operaciones individuales:

- **Uso de memoria**: WiredTiger debe mantener snapshots y registros de deshacer (_undo records_) para cada transaccion activa. Cuantas mas transacciones concurrentes, mas memoria consumen.
- **Latencia de replicacion**: Con `writeConcern: 'majority'`, el commit no se completa hasta que la mayoria de los secundarios confirman la escritura. En clusters geograficamente distribuidos, esto anyade latencia de red.
- **Conflictos de escritura**: Dos transacciones concurrentes que modifican el mismo documento generan conflictos. WiredTiger resuelve estos conflictos abortando una de las transacciones, que debe reintentarse.
- **Sobrecarga de coordinacion**: En sharded clusters, el commit en dos fases implica mensajes de coordinacion entre fragmentos que incrementan la latencia total.

> Una regla practica: si una operacion puede expresarse como una actualizacion atomica sobre un unico documento (usando `$push`, `$inc`, `$set`, arrays embebidos, etc.), debe preferirse siempre a una transaccion multi-documento. Las transacciones son el ultimo recurso, no la primera opcion.

## Buenas practicas

Mantener las transacciones cortas, minimalistas y bien acotadas es la clave para usarlas sin comprometer el rendimiento del sistema.

- **Limitar el tamano de la transaccion**: incluir solo las operaciones estrictamente necesarias. No realizar consultas pesadas ni operaciones costosas dentro de la transaccion.
- **Evitar interaccion con el usuario**: no mantener una transaccion abierta mientras se espera entrada del usuario. Toda la logica debe prepararse previamente.
- **Usar withTransaction siempre que sea posible**: delegar el reintento automatico al controlador en lugar de implementar logica de reintento manual propensa a errores.
- **Configurar writeConcern y readConcern explicitamente**: no confiar en los valores por defecto, ya que pueden variar segun la configuracion del despliegue.
- **Monitorear la tasa de conflictos**: en sistemas con alta concurrencia, la tasa de transacciones abortadas por conflicto es una metrica clave. Si es alta, reconsiderar el modelo de datos o reducir el ambito de las transacciones.
- **Preparar el esquema para transacciones**: si una coleccion participa frecuentemente en transacciones, incluir un campo de version o timestamp para detectar conflictos de forma explicita si es necesario.

