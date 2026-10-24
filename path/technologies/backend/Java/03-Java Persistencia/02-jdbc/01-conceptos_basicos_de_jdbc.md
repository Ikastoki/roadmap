# Conceptos basicos de JDBC

JDBC (Java Database Connectivity) es la API estandar del ecosistema Java para acceder a sistemas gestores de bases de datos relacionales. Forma parte del JDK desde su origen (java.sql) y se extendio con funcionalidades avanzadas en javax.sql (DataSource, ConnectionPoolDataSource, RowSet). A diferencia de otras tecnologias como ODBC (de Microsoft), JDBC es una API puramente Java que no requiere librerias nativas del sistema operativo, lo que garantiza portabilidad entre plataformas.

## Arquitectura

La arquitectura de JDBC se organiza en torno a un conjunto de interfaces core que abstraen la comunicacion con el motor de base de datos:

**DriverManager.** Es el punto de entrada clasico. Mantiene un registro de los drivers JDBC disponibles (cargados via `Class.forName()` en versiones antiguas o mediante el ServiceLoader de Java en versiones modernas). Cuando se invoca `DriverManager.getConnection(url, user, pass)`, recorre la lista de drivers registrados y devuelve el primero que acepta la URL de conexion. Es un mecanismo simple pero que no ofrece caracteristicas avanzadas como pooling o transacciones distribuidas.

**Driver.** Interfaz que establece el puente entre el DriverManager y la base de datos concreta. Cada base de datos (PostgreSQL, MySQL, Oracle, H2) proporciona su propia implementacion del interfaz `java.sql.Driver`. La JVM identifica el driver adecuado por el prefijo de la URL de conexion: `jdbc:postgresql://`, `jdbc:mysql://`, `jdbc:h2:`, etc.

**Connection.** Representa una conexion activa con la base de datos. Es el contexto en el que se ejecutan las sentencias SQL, se gestionan las transacciones y se obtienen los metadatos del catalogo. Una conexion puede estar en modo auto-commit (cada sentencia es una transaccion independiente) o en modo manual donde el programador controla explicitamente el commit y rollback. Las conexiones son recursos costosos de crear (handshake TCP, autenticacion, asignacion de memoria en el servidor), por lo que no deben crearse y destruirse por cada operacion.

**Statement.** Interfaz para ejecutar sentencias SQL estaticas. Recibe una cadena SQL y la envia tal cual al motor de base de datos. Es vulnerable a ataques de inyeccion SQL si se concatenan parametros directamente. Existen tres subtipos: `Statement` (SQL estatico), `PreparedStatement` (SQL precompilado con parametros) y `CallableStatement` (procedimientos almacenados).

**ResultSet.** Representa el conjunto de resultados de una consulta SELECT. Expone un cursor que avanza fila a fila mediante el metodo `next()` (el cursor se situa inicialmente antes de la primera fila). Permite leer valores de columna por nombre o por indice con metodos tipados: `getString()`, `getInt()`, `getDate()`, `getObject()`, entre otros.

## Paquetes java.sql y javax.sql

La API JDBC se divide en dos paquetes fundamentales:

**java.sql.** Contiene las interfaces y clases basicas: Driver, DriverManager, Connection, Statement, PreparedStatement, CallableStatement, ResultSet, SQLException, Date, Time, Timestamp, Types. Es el nucleo minimo necesario para cualquier operacion JDBC.

**javax.sql.** Contiene extensiones orientadas a entornos empresariales: DataSource (conexiones gestionadas, reemplazo preferido a DriverManager), ConnectionPoolDataSource (pool de conexiones), XADataSource (transacciones distribuidas XA), RowSet (conjuntos de datos desplazables y desconectables). Desde Java EE, el mecanismo recomendado para obtener conexiones es mediante JNDI y DataSource, no mediante DriverManager.

## Tipos de drivers JDBC

Los drivers JDBC se clasifican en cuatro tipos segun como implementan la comunicacion con la base de datos:

**Tipo 1: JDBC-ODBC Bridge.** Traduce las llamadas JDBC a ODBC, que a su vez accede al motor nativo de base de datos. Requiere una libreria ODBC instalada en el cliente. Obsoleto desde Java 8, no recomendado por su dependencia de codigo nativo y su bajo rendimiento.

**Tipo 2: Native-API/Partly Java.** Convierte llamadas JDBC a llamadas nativas de la API del cliente de la base de datos (por ejemplo, la libreria OCI de Oracle). Requiere binarios nativos instalados en el cliente. Ofrece buen rendimiento, pero sacrifica portabilidad.

**Tipo 3: Net-Protocol/All-Java.** El driver Java se comunica mediante un protocolo de red propio con un middleware (normalmente en Java), que a su vez traduce las peticiones al protocolo nativo de la base de datos. Es portable y permite centralizar la configuracion de los drivers nativos en el servidor middleware.

**Tipo 4: Native-Protocol/All-Java.** Implementa el protocolo de red nativo de la base de datos directamente en Java puro. No requiere ningun software adicional en el cliente mas alla del propio driver JAR. Es el tipo mas usado actualmente: los drivers de PostgreSQL (org.postgresql.Driver), MySQL (com.mysql.cj.jdbc.Driver), H2 (org.h2.Driver) y Oracle (oracle.jdbc.OracleDriver) son todos tipo 4. Ofrecen portabilidad total y rendimiento adecuado para la mayoria de aplicaciones.

## Driver JDBC segun la base de datos

| Base de datos | Driver | URL de conexion |
|--------------|--------|-----------------|
| PostgreSQL | org.postgresql.Driver | jdbc:postgresql://host:5432/bd |
| MySQL | com.mysql.cj.jdbc.Driver | jdbc:mysql://host:3306/bd |
| H2 | org.h2.Driver | jdbc:h2:~/bd |
| Oracle | oracle.jdbc.OracleDriver | jdbc:oracle:thin:@host:1521:bd |
| SQL Server | com.microsoft.sqlserver.jdbc.SQLServerDriver | jdbc:sqlserver://host:1433;databaseName=bd |
| SQLite | org.sqlite.JDBC | jdbc:sqlite:bd.sqlite |

El driver debe estar en el classpath del proyecto. En proyectos Maven se declara como dependencia en `pom.xml`. Sin el driver adecuado, el DriverManager lanzara una `SQLException` indicando que no se encontro un driver adecuado para la URL proporcionada.
