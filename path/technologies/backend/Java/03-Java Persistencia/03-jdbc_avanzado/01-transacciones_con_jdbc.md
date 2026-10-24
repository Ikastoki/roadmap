# Transacciones con JDBC

Una transaccion es una unidad logica de trabajo que agrupa una o mas operaciones sobre la base de datos con las propiedades ACID (Atomicity, Consistency, Isolation, Durability). JDBC proporciona control explicito sobre las transacciones a traves de los metodos de la interfaz `Connection`, permitiendo al programador decidir cuando se inicia, confirma o deshace una transaccion.

## Modo auto-commit

Por defecto, cada sentencia SQL ejecutada sobre una conexion JDBC se confirma automaticamente (auto-commit mode). Esto significa que cada `executeUpdate()` o `executeQuery()` que modifica datos es una transaccion independiente. Para agrupar varias operaciones en una misma transaccion, debe desactivarse el modo auto-commit:

```java
conn.setAutoCommit(false);
```

A partir de ese momento, todas las sentencias que se ejecuten sobre esa conexion forman parte de la misma transaccion hasta que se invoque `commit()` o `rollback()`.

## commit() y rollback()

`commit()` confirma permanentemente todas las modificaciones realizadas desde el ultimo commit o rollback. `rollback()` descarta todas las modificaciones realizadas en la transaccion actual, devolviendo la base de datos al estado anterior al inicio de la transaccion.

```java
Connection conn = null;
try {
    conn = dataSource.getConnection();
    conn.setAutoCommit(false);

    // Operacion 1: insertar pedido
    try (PreparedStatement ps1 = conn.prepareStatement(
            "INSERT INTO pedidos (cliente_id, total) VALUES (?, ?)")) {
        ps1.setLong(1, clienteId);
        ps1.setBigDecimal(2, total);
        ps1.executeUpdate();
    }

    // Operacion 2: actualizar stock
    try (PreparedStatement ps2 = conn.prepareStatement(
            "UPDATE productos SET stock = stock - ? WHERE id = ?")) {
        ps2.setInt(1, cantidad);
        ps2.setLong(2, productoId);
        ps2.executeUpdate();
    }

    // Operacion 3: insertar detalle
    try (PreparedStatement ps3 = conn.prepareStatement(
            "INSERT INTO detalle_pedido (pedido_id, producto_id, cantidad) VALUES (?, ?, ?)")) {
        ps3.setLong(1, pedidoId);
        ps3.setLong(2, productoId);
        ps3.setInt(3, cantidad);
        ps3.executeUpdate();
    }

    conn.commit(); // Confirmar todas las operaciones

} catch (SQLException e) {
    if (conn != null) {
        try {
            conn.rollback(); // Deshacer todas las operaciones
        } catch (SQLException ex) {
            logger.error("Error al hacer rollback", ex);
        }
    }
    logger.error("Error en transaccion", e);
    throw e;
} finally {
    if (conn != null) {
        conn.setAutoCommit(true); // Restaurar modo por defecto
        conn.close();
    }
}
```

> Es fundamental restaurar el modo auto-commit a `true` antes de devolver la conexion al pool. Si la conexion se devuelve con auto-commit=false, la siguiente operacion que la obtenga heredara ese estado, lo que puede provocar commits no deseados o transacciones que nunca se confirman.

## Savepoints

Los savepoints permiten marcar puntos intermedios dentro de una transaccion sobre los que se puede hacer rollback parcial sin deshacer toda la transaccion. Son utiles en operaciones complejas donde ciertos pasos pueden fallar sin invalidar el trabajo previo.

```java
conn.setAutoCommit(false);
Savepoint sp1 = null;

try {
    // Operaciones previas...
    sp1 = conn.setSavepoint("paso1");

    // Operacion que podria fallar
    try {
        realizarOperacionRiesgosa(conn);
    } catch (SQLException e) {
        if (sp1 != null) {
            conn.rollback(sp1); // Deshacer solo desde el savepoint
        }
    }

    conn.commit();

} catch (SQLException e) {
    conn.rollback(); // Deshacer todo
}
```

`releaseSavepoint(Savepoint)` libera el savepoint sin afectar a la transaccion. Una vez liberado, ya no se puede usar como destino de un rollback. No todos los motores de base de datos soportan savepoints (MySQL los soporta desde la version 5.0, H2 tambien, PostgreSQL desde siempre).

## Niveles de aislamiento

El nivel de aislamiento determina como se comporta una transaccion respecto a los cambios realizados por otras transacciones concurrentes. Los niveles definidos en JDBC son, de menor a mayor aislamiento:

**TRANSACTION_READ_UNCOMMITTED.** Permite leer datos modificados pero aun no confirmados por otras transacciones (dirty reads). El nivel mas bajo de aislamiento, apenas usado en la practica por los riesgos de consistencia que introduce.

**TRANSACTION_READ_COMMITTED.** Garantiza que solo se leen datos confirmados. Evita dirty reads pero permite non-repeatable reads (una misma consulta dentro de la misma transaccion puede devolver resultados distintos si otra transaccion modifica y confirma los datos entre ambas lecturas). Es el nivel por defecto en PostgreSQL, Oracle y SQL Server.

**TRANSACTION_REPEATABLE_READ.** Garantiza que si se lee una fila dos veces dentro de la misma transaccion, se obtiene el mismo valor. Evita dirty reads y non-repeatable reads, pero permite phantom reads (nuevas filas que aparecen en consultas con rango si otra transaccion las inserta y confirma).

**TRANSACTION_SERIALIZABLE.** El nivel mas alto de aislamiento. Ejecuta las transacciones como si fueran secuenciales, una detras de otra. Evita dirty reads, non-repeatable reads y phantom reads. El coste en rendimiento y concurrencia es elevado porque el motor debe serializar el acceso, normalmente mediante locks de tabla o bloqueos predictivos.

```java
conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);
```

El nivel de aislamiento se establece antes de iniciar la transaccion (o al menos antes de la primera sentencia). Cambiar el nivel de aislamiento durante una transaccion activa puede provocar un commit implicito en algunos motores.

| Nivel | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|-------------------|--------------|
| READ_UNCOMMITTED | Puede ocurrir | Puede ocurrir | Puede ocurrir |
| READ_COMMITTED | Evitado | Puede ocurrir | Puede ocurrir |
| REPEATABLE_READ | Evitado | Evitado | Puede ocurrir |
| SERIALIZABLE | Evitado | Evitado | Evitado |

```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);
    conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);

    // ... operaciones

    conn.commit();
}
```

> El nivel de aislamiento por defecto depende del motor de base de datos. PostgreSQL usa READ_COMMITTED por defecto; MySQL con InnoDB usa REPEATABLE_READ; Oracle usa READ_COMMITTED; H2 usa READ_COMMITTED. Siempre es recomendable establecer explicitamente el nivel de aislamiento requerido en lugar de confiar en el valor por defecto, especialmente si la aplicacion se va a desplegar sobre distintos motores de base de datos.
