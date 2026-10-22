<h1 align="center">Transacciones</h1>

<h2>📑 Contenido</h2>

- [Transacciones](#transacciones)
- [ACID](#acid)
  - [Atomicidad:](#atomicidad)
  - [Consistencia:](#consistencia)
  - [Aislamiento:](#aislamiento)
  - [Durabilidad:](#durabilidad)
- [BEGIN TRANSACTION](#begin-transaction)
- [COMMIT](#commit)
- [ROLLBACK](#rollback)
- [SAVEPOINT (no mencionado explícitamente pero relacionado)](#savepoint-no-mencionado-explícitamente-pero-relacionado)

## Transacciones

Una transacción en SQL es una unidad lógica de trabajo que contiene una o más sentencias SQL. En términos simples, una transacción significa un grupo de tareas que deben ejecutarse como una sola unidad, y se trata como una sola unidad de trabajo. Las sentencias involucradas en una transacción pueden ser una sola consulta SQL o un lote de consultas SQL1.

En el contexto de SQL Server, las transacciones son esenciales para mantener la integridad, consistencia y confiabilidad de los datos. Aquí hay algunos puntos clave sobre las transacciones en SQL Server:

## ACID

ACID significa Atomicidad, Consistencia, Aislamiento y Durabilidad.

Estas propiedades garantizan la confiabilidad y consistencia de las transacciones en una base de datos.

### Atomicidad:

Una transacción se considera atómica, lo que significa que todas las operaciones dentro de ella se ejecutan como una sola unidad o no se ejecutan en absoluto.
Si una transacción tiene éxito, todas las modificaciones de datos realizadas durante la transacción se confirman y se vuelven permanentes en la base de datos.
Si ocurre un error o se debe cancelar la transacción, todas las modificaciones de datos se deshacen.

### Consistencia:

Las transacciones garantizan que la base de datos permanezca en un estado consistente antes y después de su ejecución.
Si una transacción modifica múltiples tablas, todas las tablas deben estar en un estado coherente al final de la transacción.

### Aislamiento:

Las transacciones se ejecutan de manera aislada entre sí. Esto significa que una transacción no ve los cambios realizados por otras transacciones hasta que se complete.
El aislamiento evita problemas como lecturas sucias, lecturas no repetibles y conflictos de escritura.

### Durabilidad:

Una vez que una transacción se confirma (mediante COMMIT), sus cambios son permanentes incluso si ocurre un fallo del sistema o se reinicia el servidor.
La durabilidad garantiza que los datos modificados no se pierdan debido a fallas.

## BEGIN TRANSACTION

Marca el inicio de una transacción explícita o local.

```sql
BEGIN TRANSACTION nombre_de_la_transaccion;
```

## COMMIT

Guarda todos los cambios realizados dentro de una transacción desde el último COMMIT o ROLLBACK.

```sql
BEGIN TRANSACTION;
UPDATE Empleados SET Salario = 50000 WHERE IDEmpleado = 123;
INSERT INTO RegistroEmpleados VALUES (123, 'Salario actualizado a 50000', GETDATE());
COMMIT;
```

## ROLLBACK

Deshace todos los cambios realizados dentro de una transacción.

```sql
BEGIN TRANSACTION;
UPDATE Empleados SET Salario = 50000 WHERE IDEmpleado = 123;
-- Ocurre un error aquí, lo que cancela la transacción
ROLLBACK TRANSACTION;
```

## SAVEPOINT (no mencionado explícitamente pero relacionado)

Permite establecer un punto dentro de una transacción al que se puede volver más tarde.
Útil para deshacer parcialmente una transacción.

```sql
BEGIN TRANSACTION;
UPDATE Empleados SET Salario = 50000 WHERE IDEmpleado = 123;
SAVEPOINT MiPuntoDeGuardado;
-- Ocurre un error aquí, pero aún podemos volver al punto de guardado
ROLLBACK TO MiPuntoDeGuardado;
```
