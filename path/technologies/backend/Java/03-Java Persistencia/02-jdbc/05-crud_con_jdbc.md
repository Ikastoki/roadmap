# CRUD con JDBC

Las operaciones fundamentales de cualquier sistema de persistencia son el Create, Read, Update y Delete (CRUD). JDBC proporciona los mecanismos para ejecutar cada una de estas operaciones mediante sentencias SQL parametrizadas con `PreparedStatement`. La correcta implementacion de estas operaciones determina la integridad de los datos, la seguridad frente a inyeccion SQL y el rendimiento de la capa de acceso a datos.

## INSERT con PreparedStatement

La insercion de registros se realiza con `executeUpdate()` sobre un `PreparedStatement` que contenga una sentencia INSERT. Los valores se asignan mediante los metodos `setXxx()` correspondientes al tipo de cada columna.

```java
String sql = "INSERT INTO usuarios (nombre, email, edad, fecha_alta) VALUES (?, ?, ?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "Maria Garcia");
    ps.setString(2, "maria@example.com");
    ps.setInt(3, 30);
    ps.setDate(4, Date.valueOf(LocalDate.now()));

    int filasAfectadas = ps.executeUpdate();
    System.out.println("Filas insertadas: " + filasAfectadas);
}
```

`executeUpdate()` devuelve el numero de filas afectadas por la operacion. Para un INSERT de una sola fila, el valor esperado es 1. Si la tabla tiene triggers o constraints que impiden la insercion, el metodo lanzara una `SQLException`.

### Generacion de claves (getGeneratedKeys)

Cuando la tabla tiene una columna de tipo auto-increment (SERIAL en PostgreSQL, AUTO_INCREMENT en MySQL, IDENTITY en H2), se puede recuperar el valor generado automaticamente para la clave primaria sin necesidad de una consulta adicional:

```java
String sql = "INSERT INTO usuarios (nombre, email) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
    ps.setString(1, "Ana Lopez");
    ps.setString(2, "ana@example.com");
    ps.executeUpdate();

    try (ResultSet rs = ps.getGeneratedKeys()) {
        if (rs.next()) {
            Long idGenerado = rs.getLong(1);
            System.out.println("ID generado: " + idGenerado);
        }
    }
}
```

El flag `Statement.RETURN_GENERATED_KEYS` indica al driver que debe devolver las claves generadas. Algunos motores requieren especificar explicitamente los nombres de las columnas:

```java
try (PreparedStatement ps = conn.prepareStatement(sql, new String[]{"id"})) {
    // ...
}
```

## SELECT con ResultSet

La lectura de datos combina `executeQuery()` con el recorrido del `ResultSet`:

```java
String sql = "SELECT id, nombre, email, edad FROM usuarios WHERE activo = ? ORDER BY nombre";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setBoolean(1, true);

    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            Long id = rs.getLong("id");
            String nombre = rs.getString("nombre");
            String email = rs.getString("email");
            int edad = rs.getInt("edad");

            System.out.printf("%d | %s | %s | %d%n", id, nombre, email, edad);
        }
    }
}
```

Para consultas que devuelven una sola fila (busqueda por clave primaria), se puede evitar el bucle:

```java
if (rs.next()) {
    Usuario u = new Usuario(rs.getLong("id"), rs.getString("nombre"), ...);
} else {
    // No se encontraron resultados
}
```

## UPDATE con executeUpdate

La actualizacion de registros sigue el mismo patron que INSERT, con `executeUpdate()` devolviendo el numero de filas modificadas:

```java
String sql = "UPDATE usuarios SET email = ?, edad = ? WHERE id = ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "nuevo_email@example.com");
    ps.setInt(2, 31);
    ps.setLong(3, 1L);

    int filas = ps.executeUpdate();
    if (filas == 0) {
        System.out.println("No se encontro el usuario con id 1");
    }
}
```

Es importante verificar el valor devuelto por `executeUpdate()`: si es 0, significa que ninguna fila cumplio la condicion WHERE. Esto permite detectar actualizaciones fallidas sin necesidad de lanzar excepcion.

## DELETE con executeUpdate

La eliminacion de registros utiliza el mismo mecanismo:

```java
String sql = "DELETE FROM usuarios WHERE id = ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setLong(1, id);
    int filas = ps.executeUpdate();
    if (filas == 0) {
        System.out.println("No se encontro el usuario con id " + id);
    }
}
```

> Las operaciones DELETE sin clausula WHERE eliminaran todas las filas de la tabla. Siempre debe verificarse que la sentencia SQL contenga la condicion adecuada, especialmente cuando la consulta se construye dinamicamente. Es buena practica ejecutar primero un SELECT con la misma condicion para confirmar el alcance de la operacion.

## Batch operations

Cuando se necesita insertar, actualizar o eliminar multiples registros, la ejecucion individual de cada sentencia genera una sobrecarga de red y de compilacion SQL significativa. Las operaciones batch agrupan varias sentencias en un solo viaje de ida y vuelta al servidor:

```java
String sql = "INSERT INTO usuarios (nombre, email) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    for (Usuario u : listaUsuarios) {
        ps.setString(1, u.getNombre());
        ps.setString(2, u.getEmail());
        ps.addBatch();   // Acumula la operacion
    }

    int[] resultados = ps.executeBatch(); // Envia todo de una vez
}
```

`executeBatch()` devuelve un array de enteros donde cada elemento indica el numero de filas afectadas por cada operacion individual. Si alguna operacion falla, se lanza `BatchUpdateException` y se puede obtener el array de resultados parcial con `getUpdateCounts()`.

El metodo `addBatch()` acumula las operaciones en memoria del lado del cliente. No debe usarse con millones de registros sin controlar el tamaño del lote. Lo habitual es dividir en lotes de 50 a 100 operaciones:

```java
String sql = "INSERT INTO usuarios (nombre, email) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    int contador = 0;
    for (Usuario u : listaUsuarios) {
        ps.setString(1, u.getNombre());
        ps.setString(2, u.getEmail());
        ps.addBatch();
        contador++;

        if (contador % 100 == 0) {
            ps.executeBatch(); // Vaciar lote cada 100 registros
        }
    }
    ps.executeBatch(); // Resto de registros
}
```

## Transaccionalidad en CRUD

Por defecto, cada sentencia se ejecuta en modo auto-commit, es decir, cada `executeUpdate()` es una transaccion independiente. Para operaciones CRUD que deben ser atomicas (varias inserciones relacionadas, actualizacion con validacion previa), es necesario desactivar el auto-commit y gestionar la transaccion manualmente:

```java
conn.setAutoCommit(false);
try {
    // Operaciones CRUD...
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    throw e;
}
```

La combinacion de CRUD con transacciones y batch processing constituye la base de cualquier operacion de persistencia en JDBC. Estos mismos patrones, con las adaptaciones propias de cada tecnologia, se repiten en JPA, Hibernate y Spring Data.
