# Metadata de la base de datos

JDBC proporciona dos interfaces para explorar la estructura de la base de datos en tiempo de ejecucion: `DatabaseMetaData` para informacion del catalogo (tablas, columnas, claves, procedimientos) y `ResultSetMetaData` para informacion de las columnas de un conjunto de resultados. Ambas permiten construir herramientas de generacion de codigo, asistentes de consultas y frameworks ORM que se adaptan al esquema sin configuracion manual.

## DatabaseMetaData

Se obtiene a partir de la conexion:

```java
DatabaseMetaData meta = conn.getMetaData();
```

Proporciona informacion sobre el motor de base de datos y sus capacidades:

```java
System.out.println("Producto: " + meta.getDatabaseProductName());       // PostgreSQL
System.out.println("Version: " + meta.getDatabaseProductVersion());     // 16.0
System.out.println("Driver: " + meta.getDriverName());                  // PostgreSQL JDBC Driver
System.out.println("URL: " + meta.getURL());                            // jdbc:postgresql://...
System.out.println("Usuario: " + meta.getUserName());                   // postgres
System.out.println("Soporta transacciones: " + meta.supportsTransactions());
System.out.println("Max conexiones: " + meta.getMaxConnections());
```

### Explorar tablas: getTables()

```java
ResultSet tablas = meta.getTables(
    null,          // catalog
    "public",      // schema
    "%",           // nombre tabla (pattern)
    new String[]{"TABLE"} // tipos
);
while (tablas.next()) {
    String nombre = tablas.getString("TABLE_NAME");
    String tipo = tablas.getString("TABLE_TYPE");
    String comentario = tablas.getString("REMARKS");
    System.out.println(nombre + " (" + tipo + ")");
}
```

Los parametros admiten patrones SQL (`%` para cualquier secuencia, `_` para un caracter). Pasar `null` en catalog o schema los omite del filtro.

### Explorar columnas: getColumns()

```java
ResultSet columnas = meta.getColumns(
    null,          // catalog
    "public",      // schema
    "usuarios",    // tabla
    "%"            // patron columna
);
while (columnas.next()) {
    String nombre = columnas.getString("COLUMN_NAME");
    int tipo = columnas.getInt("DATA_TYPE");            // Types.VARCHAR
    String tipoNombre = columnas.getString("TYPE_NAME"); // varchar
    int tamano = columnas.getInt("COLUMN_SIZE");
    boolean nullable = columnas.getInt("NULLABLE") == DatabaseMetaData.columnNullable;
    String defecto = columnas.getString("COLUMN_DEF");
    int posicion = columnas.getInt("ORDINAL_POSITION");
    System.out.printf("%s %s(%d) nullable=%s def=%s%n",
        nombre, tipoNombre, tamano, nullable, defecto);
}
```

### Claves primarias: getPrimaryKeys()

```java
ResultSet pk = meta.getPrimaryKeys(null, "public", "usuarios");
while (pk.next()) {
    String columna = pk.getString("COLUMN_NAME");
    short secuencia = pk.getShort("KEY_SEQ");
    String pkName = pk.getString("PK_NAME");
    System.out.println("PK: " + columna + " (orden: " + secuencia + ")");
}
```

### Claves foraneas: getImportedKeys() / getExportedKeys()

```java
ResultSet fk = meta.getImportedKeys(null, "public", "pedidos");
while (fk.next()) {
    String colOrigen = fk.getString("FKCOLUMN_NAME");          // cliente_id
    String tablaDestino = fk.getString("PKTABLE_NAME");         // clientes
    String colDestino = fk.getString("PKCOLUMN_NAME");          // id
    String fkName = fk.getString("FK_NAME");
}
```

### Otras metadata utiles

```java
// Procedimientos almacenados
ResultSet procs = meta.getProcedures(null, "public", "%");

// Indices
ResultSet indices = meta.getIndexInfo(null, "public", "usuarios", false, true);

// Esquemas
ResultSet esquemas = meta.getSchemas();

// Tipos de datos
ResultSet tipos = meta.getTypeInfo();
```

## ResultSetMetaData

Proporciona informacion sobre las columnas del resultado de una consulta. Se obtiene a partir del `ResultSet`:

```java
try (PreparedStatement ps = conn.prepareStatement("SELECT * FROM usuarios");
     ResultSet rs = ps.executeQuery()) {

    ResultSetMetaData rsmd = rs.getMetaData();
    int numColumnas = rsmd.getColumnCount();

    for (int i = 1; i <= numColumnas; i++) {
        String nombre = rsmd.getColumnName(i);
        String tipo = rsmd.getColumnTypeName(i);
        int tamano = rsmd.getColumnDisplaySize(i);
        boolean nullable = rsmd.isNullable(i) == ResultSetMetaData.columnNullable;
        boolean autoIncrement = rsmd.isAutoIncrement(i);

        System.out.printf("Columna %d: %s (%s) tam=%d nullable=%s autoInc=%s%n",
            i, nombre, tipo, tamano, nullable, autoIncrement);
    }
}
```

Metodos disponibles:

- `getColumnCount()`: numero de columnas.
- `getColumnName(int)`: nombre de la columna.
- `getColumnLabel(int)`: alias de la columna (AS en SQL).
- `getColumnType(int)`: tipo SQL como constante de `java.sql.Types`.
- `getColumnTypeName(int)`: nombre del tipo SQL (varchar, integer, etc.).
- `getColumnDisplaySize(int)`: tamaño maximo de visualizacion.
- `isNullable(int)`: si la columna acepta NULL.
- `isAutoIncrement(int)`: si la columna es auto-increment.
- `isReadOnly(int)`: si la columna es de solo lectura.

## Dynamic SQL generation

La combinacion de `DatabaseMetaData.getColumns()` y `ResultSetMetaData` permite construir consultas SQL dinamicamente, sin conocimiento previo del esquema. Esto es util en herramientas de administracion, generadores de CRUD y frameworks ORM.

```java
public String generarSelect(String schema, String tabla) throws SQLException {
    StringBuilder sql = new StringBuilder("SELECT ");
    ResultSet columnas = meta.getColumns(null, schema, tabla, "%");
    List<String> nombres = new ArrayList<>();
    while (columnas.next()) {
        nombres.add(columnas.getString("COLUMN_NAME"));
    }
    sql.append(String.join(", ", nombres));
    sql.append(" FROM ").append(schema).append(".").append(tabla);
    return sql.toString();
}
```

```java
public String generarInsert(String tabla, ResultSetMetaData rsmd) throws SQLException {
    int cols = rsmd.getColumnCount();
    StringBuilder sql = new StringBuilder("INSERT INTO ").append(tabla).append(" (");
    StringBuilder valores = new StringBuilder(") VALUES (");
    for (int i = 1; i <= cols; i++) {
        if (rsmd.isAutoIncrement(i)) continue;
        sql.append(rsmd.getColumnName(i));
        valores.append("?");
        if (i < cols) {
            sql.append(", ");
            valores.append(", ");
        }
    }
    sql.append(valores).append(")");
    return sql.toString();
}
```

> La introspeccion de metadatos es una operacion costosa que implica consultas al catalogo del sistema. Los resultados deben cachearse (por ejemplo, en un mapa estatico `Map<String, List<ColumnaInfo>>`) en lugar de consultar cada vez. Hibernate y otros ORM cachean automaticamente esta informacion al iniciar la aplicacion.
