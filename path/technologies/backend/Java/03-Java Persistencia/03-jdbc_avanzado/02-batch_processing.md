# Batch Processing

El procesamiento por lotes (batch processing) es una tecnica que agrupa multiples sentencias SQL en un unico envio al servidor de base de datos, reduciendo drasticamente el numero de viajes de ida y vuelta (round trips) entre la aplicacion y el motor. Es especialmente util en operaciones masivas de insercion, actualizacion o eliminacion de datos.

## Funcionamiento basico

El mecanismo batch en JDBC se construye sobre `Statement` o `PreparedStatement` mediante tres metodos:

- `addBatch(String sql)` o `addBatch()`: anade una sentencia al lote actual.
- `executeBatch()`: envia todas las sentencias acumuladas al servidor en un unico viaje.
- `clearBatch()`: vacia el lote sin ejecutarlo.

```java
String sql = "INSERT INTO registros (codigo, valor) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    for (int i = 0; i < 1000; i++) {
        ps.setString(1, "COD-" + i);
        ps.setInt(2, i * 10);
        ps.addBatch();
    }

    int[] resultados = ps.executeBatch();
}
```

`executeBatch()` devuelve un array de enteros donde cada elemento indica el numero de filas afectadas por la operacion correspondiente en el lote. Los valores pueden ser:

- Un entero positivo: numero de filas afectadas.
- `Statement.SUCCESS_NO_INFO`: la operacion se ejecuto correctamente pero se desconoce el numero de filas.
- `Statement.EXECUTE_FAILED`: la operacion fallo (solo si se continua tras error).

## executeLargeBatch()

Desde Java 8, cuando el numero de filas afectadas por una operacion supera `Integer.MAX_VALUE`, se debe usar `executeLargeBatch()` que devuelve un array de `long`:

```java
long[] resultados = ps.executeLargeBatch();
```

## Batch con transacciones

La combinacion de batch processing con transacciones controladas manualmente ofrece el maximo rendimiento. Al desactivar el auto-commit, todas las operaciones del lote se ejecutan dentro de la misma transaccion, lo que permite hacer rollback completo si algo falla y, ademas, reduce la sobrecarga de confirmacion por operacion.

```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);

    String sql = "INSERT INTO logs (fecha, mensaje) VALUES (?, ?)";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        for (LogEntry log : logs) {
            ps.setTimestamp(1, log.getFecha());
            ps.setString(2, log.getMensaje());
            ps.addBatch();
        }
        ps.executeBatch();
    }

    conn.commit();

} catch (SQLException e) {
    conn.rollback();
    throw e;
}
```

## Manejo de errores en batch

Si una de las operaciones del lote falla, se lanza `BatchUpdateException`. El metodo `getUpdateCounts()` devuelve los resultados de las operaciones que se ejecutaron antes del fallo:

```java
try {
    ps.executeBatch();
} catch (BatchUpdateException e) {
    int[] resultados = e.getUpdateCounts();
    for (int i = 0; i < resultados.length; i++) {
        if (resultados[i] == Statement.EXECUTE_FAILED) {
            System.err.println("Operacion " + i + " fallo");
        } else {
            System.out.println("Operacion " + i + " afecto " + resultados[i] + " filas");
        }
    }
}
```

## Rendimiento

El batch processing mejora el rendimiento de forma dramatica al reducir los round trips. Con JDBC sin batch, insertar 1000 registros implica 1000 envios individuales al servidor. Con batch, es un unico envio con 1000 conjuntos de parametros.

| Operacion | Sin batch | Con batch (tamano 100) | Con batch (tamano 1000) |
|-----------|-----------|----------------------|------------------------|
| 1000 inserts | ~5000 ms | ~200 ms | ~80 ms |
| 10000 inserts | ~45000 ms | ~1500 ms | ~600 ms |
| 100000 inserts | ~420000 ms | ~12000 ms | ~5000 ms |

Las cifras son orientativas y dependen del motor de base de datos, latencia de red y configuracion del driver. La mejora tipica esta entre 10x y 100x respecto a la ejecucion individual.

## Batch size optimo

El tamaño del lote influye en el rendimiento y en el consumo de memoria. Un lote demasiado pequeno no aprovecha al maximo la reduccion de round trips. Un lote demasiado grande puede saturar la memoria del cliente (los parametros se acumulan en memoria hasta que se ejecuta el batch) y puede provocar timeouts en el servidor.

Los valores recomendados estan entre 50 y 100 operaciones por lote. Algunas guias de rendimiento sugieren hasta 500, pero partir de 50-100 y medir es la estrategia correcta:

```java
int batchSize = 100;
int count = 0;

for (Registro r : registros) {
    ps.setString(1, r.getCodigo());
    ps.setInt(2, r.getValor());
    ps.addBatch();
    count++;

    if (count % batchSize == 0) {
        ps.executeBatch();
        ps.clearBatch();
    }
}

// Resto de registros
if (count % batchSize != 0) {
    ps.executeBatch();
}
```

## Batch con Statement (sin parametros)

Aunque menos comun, tambien es posible usar batch con `Statement` para sentencias SQL que no requieren parametrizacion:

```java
try (Statement stmt = conn.createStatement()) {
    stmt.addBatch("INSERT INTO logs VALUES (1, 'inicio')");
    stmt.addBatch("INSERT INTO logs VALUES (2, 'proceso')");
    stmt.addBatch("INSERT INTO logs VALUES (3, 'fin')");
    int[] resultados = stmt.executeBatch();
}
```

> No todas las bases de datos soportan batch processing de la misma forma. MySQL requiere la propiedad `rewriteBatchedStatements=true` en la URL de conexion para que los batches de PreparedStatement se reescriban como una unica sentencia INSERT con multiples VALUES. Sin esta propiedad, MySQL ejecuta los batches como sentencias individuales, anulando la ventaja de rendimiento. PostgreSQL y H2 soportan batch de forma nativa sin configuracion adicional.
