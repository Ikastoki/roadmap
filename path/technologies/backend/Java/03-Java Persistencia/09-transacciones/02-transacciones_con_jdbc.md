# Transacciones con JDBC

JDBC proporciona el nivel mas bajo de control transaccional en Java. Trabajar directamente con la conexion JDBC permite un control granular sobre los limites de la transaccion, los puntos de salvaguarda y el comportamiento de aislamiento. Es la base sobre la que se construyen todos los mecanismos transaccionales de mas alto nivel.

## setAutoCommit

Por defecto, las conexiones JDBC operan en modo autocommit: cada sentencia SQL se ejecuta y confirma inmediatamente como una transaccion independiente. Para agrupar multiples operaciones en una sola transaccion, es necesario desactivar este modo.

```java
Connection conn = dataSource.getConnection();
try {
    conn.setAutoCommit(false);   // Inicia una transaccion explicita

    try (PreparedStatement ps1 = conn.prepareStatement(
            "INSERT INTO cuentas (id, saldo) VALUES (?, ?)")) {
        ps1.setInt(1, 1);
        ps1.setBigDecimal(2, new BigDecimal("1000.00"));
        ps1.executeUpdate();
    }

    try (PreparedStatement ps2 = conn.prepareStatement(
            "UPDATE cuentas SET saldo = saldo - ? WHERE id = ?")) {
        ps2.setBigDecimal(1, new BigDecimal("100.00"));
        ps2.setInt(2, 1);
        ps2.executeUpdate();
    }

    conn.commit();                // Confirma la transaccion
} catch (SQLException e) {
    conn.rollback();              // Revierte todos los cambios
    throw e;
} finally {
    conn.close();
}
```

## commit y rollback

`Connection.commit()` hace permanentes todos los cambios realizados desde el ultimo `commit()` o `rollback()`. `Connection.rollback()` descarta todos los cambios y restaura el estado anterior.

```java
public void transferir(int desde, int hacia, BigDecimal monto) throws SQLException {
    Connection conn = null;
    try {
        conn = dataSource.getConnection();
        conn.setAutoCommit(false);

        String debito = "UPDATE cuentas SET saldo = saldo - ? WHERE id = ?";
        try (PreparedStatement ps = conn.prepareStatement(debito)) {
            ps.setBigDecimal(1, monto);
            ps.setInt(2, desde);
            if (ps.executeUpdate() != 1) {
                throw new SQLException("Cuenta origen no encontrada");
            }
        }

        String credito = "UPDATE cuentas SET saldo = saldo + ? WHERE id = ?";
        try (PreparedStatement ps = conn.prepareStatement(credito)) {
            ps.setBigDecimal(1, monto);
            ps.setInt(2, hacia);
            if (ps.executeUpdate() != 1) {
                throw new SQLException("Cuenta destino no encontrada");
            }
        }

        conn.commit();
    } catch (SQLException e) {
        if (conn != null) {
            try {
                conn.rollback();
            } catch (SQLException ex) {
                e.addSuppressed(ex);
            }
        }
        throw e;
    } finally {
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                // Log y continuar
            }
        }
    }
}
```

> El bloque `finally` debe cerrar la conexion independientemente del resultado. No cerrar la conexion en caso de excepcion provoca fugas de recursos en el pool.

## Savepoints

Los savepoints permiten establecer puntos intermedios dentro de una transaccion. Se puede realizar un rollback parcial hasta un savepoint sin deshacer toda la transaccion.

```java
conn.setAutoCommit(false);

Savepoint sp1 = conn.setSavepoint("insertados");

try {
    // Primera operacion
    insertarCliente(conn, cliente);

    Savepoint sp2 = conn.setSavepoint("direccion");
    try {
        insertarDireccion(conn, cliente.getId(), direccion);
    } catch (SQLException e) {
        conn.rollback(sp2);   // Revierte solo la direccion
        insertarDireccion(conn, cliente.getId(), direccionAlternativa);
    }

    conn.commit();
} catch (SQLException e) {
    conn.rollback(sp1);       // Revierte hasta el savepoint "insertados"
    throw e;
}
```

Los savepoints solo estan disponibles si la base de datos los soporta. Se puede verificar con `DatabaseMetaData.supportsSavepoints()`.

> El uso excesivo de savepoints puede degradar el rendimiento. Cada savepoint consume recursos en el servidor de base de datos hasta que se confirma o revierte la transaccion.

## Timeout de transaccion

JDBC permite establecer un tiempo maximo de ejecucion para una transaccion mediante `setTransactionTimeout()` en la capa de `javax.sql.XAConnection`, o a traves del metodo `setQueryTimeout()` en Statement.

```java
Statement stmt = conn.createStatement();
stmt.setQueryTimeout(30);  // 30 segundos como maximo
ResultSet rs = stmt.executeQuery("SELECT * FROM pedidos WHERE ...");
```

Un timeout evita que una consulta mal optimizada o un bloqueo por concurrencia mantengan la transaccion abierta indefinidamente, consumiendo recursos del pool y de la base de datos.

## Transacciones de solo lectura

Marcar una transaccion como de solo lectura permite a la base de datos optimizar internamente la ejecucion, evitando bloqueos de escritura y reduciendo la sobrecarga de gestion transaccional.

```java
conn.setReadOnly(true);
conn.setAutoCommit(false);  // Necesario aunque sea solo lectura

try (Statement stmt = conn.createStatement()) {
    ResultSet rs = stmt.executeQuery("SELECT count(*) FROM pedidos");
    // ...
    conn.commit();          // Commit necesario aunque no haya modificaciones
}
```

> No todas las bases de datos responden igual a `setReadOnly(true)`. En PostgreSQL, habilita optimizaciones como la eliminacion de bloqueos de escritura. En H2, el comportamiento depende del modo de compatibilidad.

## Manejo de excepciones y rollback

Cuando se produce una `SQLException`, no todas las excepciones permiten continuar la transaccion. La clase `SQLException` tiene un metodo `getSQLState()` que devuelve el codigo de estado SQL estándar (SQLSTATE).

```java
try {
    conn.setAutoCommit(false);
    // operaciones...
    conn.commit();
} catch (SQLException e) {
    conn.rollback();

    switch (e.getSQLState()) {
        case "40001":  // Serialization failure
            // Reintentar la transaccion
            break;
        case "23505":  // Unique violation
            // Error de duplicado, no reintentar
            throw new DatosDuplicadosException(e);
        default:
            throw new BaseDeDatosException(e);
    }
}
```

| SQLSTATE | Significado | Accion recomendada |
|----------|-------------|-------------------|
| 40001 | Fallo de serializacion | Reintentar transaccion |
| 40003 | Conexion perdida | Reintentar transaccion |
| 23505 | Violacion de unicidad | No reintentar |
| 23000 | Violacion de integridad | No reintentar |
