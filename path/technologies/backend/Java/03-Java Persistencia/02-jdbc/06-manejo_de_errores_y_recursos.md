# Manejo de errores y recursos

La gestion de errores y el cierre correcto de recursos son dos aspectos criticos en JDBC que diferencian un codigo robusto de uno propenso a fugas de conexiones y comportamientos impredecibles. Una conexion que no se cierra agota el pool de conexiones del servidor; un ResultSet que no se libera retiene memoria y bloqueos; una excepcion mal manejada puede dejar la base de datos en un estado inconsistente.

## SQLException

Todas las operaciones JDBC lanzan `SQLException` cuando ocurre un error relacionado con la base de datos. A diferencia de otras excepciones en Java, `SQLException` es una excepcion checked (verificada), lo que obliga al programador a declararla o capturarla.

```java
try {
    // operaciones JDBC
} catch (SQLException e) {
    System.err.println("Mensaje: " + e.getMessage());
    System.err.println("Codigo SQL: " + e.getErrorCode());
    System.err.println("Estado SQL: " + e.getSQLState());
}
```

**SQLState.** Es un codigo estandar de cinco caracteres definido por el estandar SQL (ANSI/ISO). Los dos primeros caracteres indican la clase del error: `22` para violacion de integridad de datos, `23` para violacion de restriccion de integridad, `42` para error de sintaxis SQL. El codigo `00000` indica exito.

**ErrorCode.** Es un codigo especifico del driver o del motor de base de datos, no portable entre distintos fabricantes. En PostgreSQL, `0` indica ausencia de error; en MySQL, el codigo corresponde al error nativo del servidor.

**Encadenamiento.** Una `SQLException` puede contener una cadena de excepciones encadenadas mediante `getNextException()`. Esto ocurre cuando se producen multiples errores en una misma operacion, por ejemplo, en una ejecucion batch:

```java
catch (SQLException e) {
    while (e != null) {
        System.err.println(e.getMessage());
        e = e.getNextException();
    }
}
```

## Checked vs unchecked

`SQLException` es checked, lo que obliga a capturarla o declararla en la firma del metodo. Esto ha sido objeto de debate durante anos porque fuerza al programador a tratar el error en el punto de llamada, cuando a menudo la capa de presentacion no puede hacer nada con un error de base de datos salvo mostrar un mensaje generico.

Las alternativas para evitar la propagacion de `SQLException` a traves de toda la pila de llamadas son:

1. **Traduccion a excepcion no verificada.** Envolver `SQLException` en una `RuntimeException` propia (por ejemplo, `DataAccessException`) que las capas superiores capturan de forma centralizada.

2. **Spring JDBC Template.** Spring traduce automaticamente `SQLException` a su jerarquia de `DataAccessException` (no verificada) proporcionando mensajes descriptivos y codigos de error portables.

3. **DAO con excepcion propia.** Definir una excepcion de capa de persistencia que envuelva la causa original:

```java
public class PersistenciaException extends RuntimeException {
    public PersistenciaException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## try-with-resources

Java 7 introdujo el bloque try-with-resources como la forma correcta de gestionar recursos que implementan `AutoCloseable`. Connection, Statement y ResultSet implementan esta interfaz, por lo que deben cerrarse siempre dentro de un try-with-resources:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {

    while (rs.next()) {
        // procesar resultados
    }

} catch (SQLException e) {
    logger.error("Error en consulta", e);
    throw new PersistenciaException("Error al obtener usuarios", e);
}
```

El orden de cierre es inverso al de declaracion: primero `rs`, luego `ps`, finalmente `conn`. Si ocurre una excepcion durante el cierre, esta se suprime y la excepcion original (la que motivo la salida del bloque try) se propaga. Las excepciones suprimidas pueden recuperarse con `Throwable.getSuppressed()`.

```java
catch (SQLException e) {
    Throwable[] suprimidas = e.getSuppressed();
    for (Throwable s : suprimidas) {
        logger.warn("Error suprimido durante cierre", s);
    }
}
```

> En Java 9 y posteriores, las variables declaradas fuera del try-with-resources pueden usarse dentro de el siempre que sean efectivamente finales (effectively final). Esto permite tener la conexion fuera del bloque y reutilizarla en varias operaciones.

## Cerrar Connection, Statement y ResultSet

Cuando no se usa try-with-resources (por ejemplo, cuando la conexion se mantiene abierta para varias operaciones), debe garantizarse el cierre de cada recurso en un bloque finally:

```java
Connection conn = null;
PreparedStatement ps = null;
ResultSet rs = null;

try {
    conn = dataSource.getConnection();
    ps = conn.prepareStatement(sql);
    rs = ps.executeQuery();
    // procesar...
} catch (SQLException e) {
    logger.error("Error", e);
} finally {
    if (rs != null) {
        try { rs.close(); } catch (SQLException e) { /* ignorar */ }
    }
    if (ps != null) {
        try { ps.close(); } catch (SQLException e) { /* ignorar */ }
    }
    if (conn != null) {
        try { conn.close(); } catch (SQLException e) { /* ignorar */ }
    }
}
```

Cada cierre debe ir en su propio bloque try-catch para que el fallo en un cierre no impida el cierre de los demas. No es necesario cerrar explicitamente Statement y ResultSet si se cierra la Connection (el driver los cierra en cascada), pero la especificacion JDBC recomienda hacerlo explicitamente para liberar recursos del servidor lo antes posible.

## Patrones de cierre seguro

**Patron 1: try-with-resources anidado.** Cuando la conexion se obtiene de un pool y se devuelve al pool al cerrarla, se pueden anidar bloques try-with-resources para controlar el ambito de cada recurso:

```java
try (Connection conn = dataSource.getConnection()) {
    // Primera operacion
    try (PreparedStatement ps = conn.prepareStatement(sql1);
         ResultSet rs = ps.executeQuery()) {
        // ...
    }
    // Segunda operacion
    try (PreparedStatement ps = conn.prepareStatement(sql2)) {
        ps.executeUpdate();
    }
} catch (SQLException e) {
    logger.error("Error en transaccion", e);
}
```

**Patron 2: metodo helper de cierre.** Para codigo legacy o situaciones donde no se puede usar try-with-resources, un metodo estatico que cierra recursos de forma segura:

```java
public static void closeQuietly(AutoCloseable resource) {
    if (resource != null) {
        try {
            resource.close();
        } catch (Exception e) {
            // Loggear y continuar
        }
    }
}
```

**Patron 3: Apache Commons DbUtils.** La libreria `commons-dbutils` proporciona `DbUtils.closeQuietly()` que maneja el cierre seguro de conexiones, statements y resultsets, incluyendo la supresion de excepciones durante el cierre.
