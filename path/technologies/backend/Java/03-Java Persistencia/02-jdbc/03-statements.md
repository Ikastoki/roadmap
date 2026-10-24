# Statements

JDBC define tres tipos de objetos para ejecutar sentencias SQL: `Statement`, `PreparedStatement` y `CallableStatement`. Cada uno esta disenado para un escenario concreto, y la eleccion entre ellos afecta tanto a la seguridad como al rendimiento y la legibilidad del codigo.

## Statement

`Statement` es la interfaz mas basica para ejecutar SQL estatico. Recibe una cadena SQL completa y la envia directamente al motor de base de datos, que la compila y ejecuta. Es apropiado para sentencias SQL que no cambian entre ejecuciones y que no incluyen parametros variables, como consultas a tablas del sistema o DDL (CREATE, ALTER, DROP).

```java
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM pg_tables WHERE schemaname = 'public'");
```

Los metodos principales son `executeQuery(String)` para SELECT (devuelve ResultSet), `executeUpdate(String)` para INSERT/UPDATE/DELETE (devuelve int con filas afectadas) y `execute(String)` para sentencias cuyo resultado es desconocido (puede devolver boolean indicando si hay ResultSet).

El problema fundamental de `Statement` es la vulnerabilidad a inyeccion SQL. Al concatenar valores directamente en la cadena SQL, un atacante puede manipular el sentido de la consulta introduciendo caracteres de escape:

```java
// PELIGROSO: concatenacion directa
String sql = "SELECT * FROM usuarios WHERE nombre = '" + nombre + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);
```

Si `nombre` vale `' OR '1'='1`, la consulta se convierte en `SELECT * FROM usuarios WHERE nombre = '' OR '1'='1'`, devolviendo todos los registros. Por esta razon, `Statement` solo debe usarse para SQL sin interpolacion de variables de usuario.

## PreparedStatement

`PreparedStatement` extiende `Statement` y soluciona los problemas de seguridad y rendimiento de su predecesor. La sentencia SQL se escribe con parametros marcados con `?` (placeholders) y se precompila en el motor de base de datos. En cada ejecucion solo se envian los valores de los parametros, no la sentencia completa.

```java
String sql = "SELECT * FROM usuarios WHERE nombre = ? AND edad > ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "Ana");
    ps.setInt(2, 18);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            System.out.println(rs.getString("nombre"));
        }
    }
}
```

**Prevencion de inyeccion SQL.** Los parametros se envian por separado del SQL, y el driver los escapa automaticamente. El contenido del parametro nunca se interpreta como parte de la sintaxis SQL, lo que elimina por completo el riesgo de inyeccion (siempre que no se concatenen valores directamente, que es justo lo que se debe evitar).

**Metodos de parametrizacion.** Cada tipo SQL tiene su correspondiente metodo `setXxx()`:

```java
ps.setString(int index, String value);       // VARCHAR, TEXT
ps.setInt(int index, int value);             // INTEGER
ps.setLong(int index, long value);           // BIGINT
ps.setDouble(int index, double value);       // DOUBLE, FLOAT
ps.setBigDecimal(int index, BigDecimal val); // DECIMAL, NUMERIC
ps.setBoolean(int index, boolean value);     // BOOLEAN
ps.setDate(int index, java.sql.Date value);  // DATE
ps.setTimestamp(int index, Timestamp value); // TIMESTAMP
ps.setObject(int index, Object value);       // conversion automatica
ps.setNull(int index, int sqlType);          // valores NULL
```

**Precompilacion y rendimiento.** En la primera ejecucion, el motor de base de datos compila la sentencia y almacena el plan de ejecucion. En ejecuciones posteriores con distintos parametros, el plan se reutiliza, ahorrando el coste de compilacion. Esto supone una mejora significativa en operaciones repetitivas (INSERTS en bucle, mismo SELECT con distintos IDs).

**Batch con PreparedStatement.** `PreparedStatement` soporta la ejecucion por lotes:

```java
String sql = "INSERT INTO usuarios(nombre, email) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    for (Usuario u : usuarios) {
        ps.setString(1, u.getNombre());
        ps.setString(2, u.getEmail());
        ps.addBatch();
    }
    int[] resultados = ps.executeBatch();
}
```

## CallableStatement

`CallableStatement` extiende `PreparedStatement` y permite invocar procedimientos almacenados y funciones de la base de datos. Los procedimientos pueden tener parametros de entrada (IN), de salida (OUT) y de entrada/salida (INOUT).

```java
// Procedimiento: obtener_usuario(IN id INT, OUT nombre VARCHAR, OUT email VARCHAR)
String sql = "{call obtener_usuario(?, ?, ?)}";
try (CallableStatement cs = conn.prepareCall(sql)) {
    cs.setInt(1, 42);                    // IN
    cs.registerOutParameter(2, Types.VARCHAR);  // OUT
    cs.registerOutParameter(3, Types.VARCHAR);  // OUT
    cs.execute();
    String nombre = cs.getString(2);
    String email = cs.getString(3);
}
```

El soporte para procedimientos almacenados depende de cada motor de base de datos. PostgreSQL usa `select * from funcion(?, ?)` o `call procedimiento(?, ?)` segun la version. MySQL y Oracle usan `{call procedure_name(?, ?)}`. CallableStatement tambien permite ejecutar funciones que devuelven un ResultSet.

## Diferencias y cuando usar cada uno

| Aspecto | Statement | PreparedStatement | CallableStatement |
|---------|-----------|------------------|-------------------|
| SQL estatico | Si | Si | Si |
| Parametros | No (concatenacion) | Si (placeholders ?) | Si (IN/OUT/INOUT) |
| Seguridad inyeccion | No | Si | Si |
| Precompilacion | No | Si | Si |
| Batch | Si | Si | Limitado |
| Procedimientos | No (manualmente) | No | Si |
| Rendimiento repetitivo | Bajo | Alto | Alto |

**Statement** solo debe usarse para SQL generado dinamicamente sin valores de usuario, como consultas de metadata (listar tablas, esquemas) o DDL. No se recomienda para operaciones sobre datos de usuario.

**PreparedStatement** es la opcion por defecto para cualquier sentencia SQL que involucre datos variables. Ofrece seguridad contra inyeccion SQL, precompilacion que mejora el rendimiento en ejecuciones repetidas, y codigo mas limpio al separar la estructura SQL de los valores.

**CallableStatement** se usa exclusivamente para invocar procedimientos almacenados o funciones de base de datos. Su uso presupone que existe logica de negocio implementada en el motor de base de datos (stored procedures), lo cual es una decision arquitectonica que tiene defensores (centralizacion, rendimiento) y detractores (acoplamiento al motor, dificultad de testing).

> La regla practica es la siguiente: si la sentencia SQL contiene algun valor variable, debe usarse `PreparedStatement`. Cualquier excepcion debe estar justificada por un motivo muy concreto (generacion de SQL dinamico para busquedas con filtros variables, por ejemplo). El uso de `Statement` con valores concatenados se considera una mala practica de seguridad en cualquier aplicacion profesional.
