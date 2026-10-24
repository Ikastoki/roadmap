# ResultSet

ResultSet es la interfaz de JDBC que representa el conjunto de resultados devuelto por una consulta SELECT. Expone un cursor que permite recorrer las filas obtenidas y acceder a los valores de cada columna mediante metodos tipados. Es, junto con Connection y Statement, uno de los tres recursos fundamentales de JDBC que deben gestionarse cuidadosamente para evitar fugas de recursos.

## Cursor y navegacion

El cursor de un ResultSet se situa inicialmente antes de la primera fila (posicion BOF, begin of file). Para avanzar a la primera fila y a las sucesivas se utiliza el metodo `next()`, que devuelve `true` mientras haya filas disponibles y `false` cuando se alcanza el final del conjunto.

```java
try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        // Procesar fila actual
    }
}
```

Cuando el ResultSet es de tipo desplazable (scrollable), se dispone de metodos adicionales de navegacion:

```java
rs.next();          // Avanza a la siguiente fila
rs.previous();      // Retrocede a la fila anterior
rs.first();         // Se situa en la primera fila
rs.last();          // Se situa en la ultima fila
rs.absolute(10);    // Se situa en la fila 10
rs.relative(-5);    // Retrocede 5 filas desde la posicion actual
rs.beforeFirst();   // Vuelve a la posicion inicial (antes de la primera)
rs.afterLast();     // Se situa despues de la ultima fila
```

El metodo `getRow()` devuelve el numero de la fila actual (1-indexed). `isBeforeFirst()`, `isFirst()`, `isLast()` e `isAfterLast()` permiten consultar la posicion del cursor. El metodo `last()` seguido de `getRow()` es una forma de conocer el tamaño total del resultado sin necesidad de una consulta COUNT adicional, aunque obliga al motor a materializar el resultado completo, lo que puede ser costoso.

## Lectura de valores

Para acceder a los valores de la fila actual se usan metodos `getXxx()` que aceptan el indice de columna (empezando en 1) o el nombre de la columna:

```java
String nombre = rs.getString("nombre");       // Por nombre
int edad = rs.getInt("edad");
String email = rs.getString(3);                // Por indice
```

Metodos tipados principales:

```java
rs.getString(columnIndex);      // VARCHAR, CHAR, TEXT
rs.getInt(columnIndex);         // INTEGER, SMALLINT
rs.getLong(columnIndex);        // BIGINT
rs.getDouble(columnIndex);      // DOUBLE, FLOAT, REAL
rs.getBigDecimal(columnIndex);  // DECIMAL, NUMERIC
rs.getBoolean(columnIndex);     // BOOLEAN, BIT
rs.getDate(columnIndex);        // DATE (java.sql.Date)
rs.getTime(columnIndex);        // TIME (java.sql.Time)
rs.getTimestamp(columnIndex);   // TIMESTAMP (java.sql.Timestamp)
rs.getBytes(columnIndex);       // BLOB, BINARY
rs.getObject(columnIndex);      // Cualquier tipo (conversion generica)
```

`getObject()` devuelve el valor como `Object`. El tipo concreto depende del mapeo entre el tipo SQL y Java definido por el driver. Es util cuando no se conoce el tipo de columna en tiempo de compilacion, pero obliga a hacer un casting posterior que puede lanzar ClassCastException.

> Los indices de columna empiezan en 1, no en 0. Es un error comun usar `rs.getString(0)` que lanza SQLException con el mensaje "Column index out of range".

## Manejo de valores NULL

Los metodos `getXxx()` devuelven `null` para tipos de objeto (String, BigDecimal, Date) cuando la columna es NULL en la base de datos. Para tipos primitivos (int, long, double, boolean), devuelven 0 o false, lo que impide distinguir entre un valor 0 real y un NULL. Para detectar NULLs correctamente debe usarse `wasNull()` inmediatamente despues de la lectura:

```java
int edad = rs.getInt("edad");
if (rs.wasNull()) {
    // La columna era NULL en base de datos
    edad = -1; // Valor por defecto
}
```

## Tipos de ResultSet

El tipo de un ResultSet se define al crear el Statement mediante los parametros `resultSetType` y `resultSetConcurrency`:

```java
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_READ_ONLY
);
```

**ResultSet.TYPE_FORWARD_ONLY.** El cursor solo puede avanzar con `next()`. Es el tipo por defecto y el mas eficiente. No permite `previous()`, `first()`, `last()` ni `absolute()`. El motor de base de datos puede optimizar la entrega de resultados sin necesidad de almacenar filas adicionales en memoria.

**ResultSet.TYPE_SCROLL_INSENSITIVE.** Permite navegacion bidireccional y posicionamiento absoluto. El ResultSet es insensible a los cambios realizados por otras transacciones despues de la apertura del cursor: la vista de los datos es fija en el momento de la consulta. El driver normalmente materializa el resultado completo para permitir el desplazamiento, lo que puede consumir mucha memoria.

**ResultSet.TYPE_SCROLL_SENSITIVE.** Permite navegacion bidireccional y refleja los cambios realizados por otras transacciones. El soporte para este tipo es limitado y depende del motor de base de datos. Muchos drivers lo tratan como SCROLL_INSENSITIVE o directamente lanzan una excepcion.

**ResultSet.CONCUR_READ_ONLY.** El ResultSet no puede modificarse. Es el modo por defecto y el unico soportado de forma universal.

**ResultSet.CONCUR_UPDATABLE.** Permite actualizar, insertar y eliminar filas directamente sobre el ResultSet mediante `updateXxx()`, `updateRow()`, `insertRow()` y `deleteRow()`. Requiere que la consulta seleccione la clave primaria y que el driver soporte la operacion. Es poco usado en la practica porque el enfoque con sentencias UPDATE explicitas es mas claro y portable.

```java
try (Statement stmt = conn.createStatement(
        ResultSet.TYPE_FORWARD_ONLY,
        ResultSet.CONCUR_READ_ONLY);
     ResultSet rs = stmt.executeQuery("SELECT * FROM usuarios")) {

    while (rs.next()) {
        System.out.printf("ID: %d, Nombre: %s%n",
            rs.getInt("id"),
            rs.getString("nombre"));
    }
}
```

> Para la mayoria de aplicaciones, `TYPE_FORWARD_ONLY` con `CONCUR_READ_ONLY` es la combinacion adecuada. Los ResultSet desplazables son utiles en interfaces de usuario con paginacion (listas que muestran N registros y permiten avanzar/retroceder), pero el coste de memoria asociado a materializar el resultado puede ser elevado. En esos casos, suele ser mejor opcion paginar a nivel SQL (LIMIT/OFFSET) en lugar de usar ResultSet desplazables.
