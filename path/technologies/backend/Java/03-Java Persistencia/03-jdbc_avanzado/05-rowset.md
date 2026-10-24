# RowSet

La interfaz `RowSet` extiende `ResultSet` y anade funcionalidades que el ResultSet estandar no ofrece: soporte para Java Beans (propiedades con getters/setters), eventos de cambio de datos, capacidad de operar desconectado del origen de datos (disconnected) y serializacion para entornos distribuidos. Forma parte del paquete `javax.sql.rowset` desde Java 1.4.

A diferencia de un ResultSet convencional, que requiere una conexion abierta durante todo su ciclo de vida, los RowSet desconectados pueden obtener los datos, cerrar la conexion, modificarlos localmente y reconectarse para aplicar los cambios. Esto los hace especialmente utiles en aplicaciones moviles, clientes ricos (desktop) y entornos con conectividad intermitente.

## Tipos de RowSet

La especificacion define cinco implementaciones estandar, cada una disenada para un caso de uso concreto:

### CachedRowSet

`CachedRowSet` es el tipo mas versatil. Obtiene los datos de la base de datos, los almacena en memoria como un conjunto de filas desconectadas y cierra la conexion. Las operaciones de lectura y navegacion trabajan contra la cache en memoria, sin necesidad de conexion. Cuando se desean persistir los cambios, se reconecta y aplica las modificaciones.

```java
CachedRowSet crs = new CachedRowSetImpl(); // o RowSetProvider.newInstance()
crs.setUrl("jdbc:postgresql://localhost:5432/mi_bd");
crs.setUsername("postgres");
crs.setPassword("postgres");
crs.setCommand("SELECT id, nombre, email FROM usuarios WHERE activo = ?");
crs.setInt(1, 1);
crs.execute(); // Aqui se abre conexion, se ejecuta la consulta y se cierra la conexion

// Ya desconectado: navegar y leer
while (crs.next()) {
    System.out.println(crs.getString("nombre"));
}

// Modificar datos (sin conexion)
crs.beforeFirst();
while (crs.next()) {
    if (crs.getString("email").endsWith("@antiguo.com")) {
        crs.updateString("email",
            crs.getString("email").replace("@antiguo.com", "@nuevo.com"));
        crs.updateRow();
    }
}

// Reconectar y aplicar cambios
crs.acceptChanges(); // Vuelve a conectar y ejecuta UPDATEs
```

`CachedRowSet` soporta todas las operaciones de navegacion heredadas de ResultSet (next, previous, first, last, absolute, relative) sin necesidad de que el ResultSet original sea desplazable, porque los datos estan completamente en memoria.

Sus puntos debiles son el consumo de memoria (todos los datos se cargan en RAM del cliente) y la ausencia de control de concurrencia optimista: al llamar a `acceptChanges()`, las actualizaciones pueden sobrescribir cambios realizados por otros usuarios si no se implementa un mecanismo de versionado.

### WebRowSet

`WebRowSet` extiende `CachedRowSet` y anade la capacidad de exportar e importar el conjunto de datos en formato XML. Esta disenado para la transmision de datos entre capas o sistemas heterogeneos a traves de servicios web.

```java
WebRowSet wrs = new WebRowSetImpl();
wrs.setUrl("jdbc:postgresql://localhost:5432/mi_bd");
wrs.setCommand("SELECT * FROM productos");
wrs.execute();

// Exportar a XML
try (FileWriter writer = new FileWriter("productos.xml")) {
    wrs.writeXml(writer);
}

// Importar desde XML
WebRowSet wrs2 = new WebRowSetImpl();
try (FileReader reader = new FileReader("productos.xml")) {
    wrs2.readXml(reader);
}
```

El XML generado incluye tanto los datos como los metadatos (nombres de columna, tipos, valores nulos), lo que permite reconstruir completamente el RowSet en el destino. Es compatible con XSLT para transformacion de datos.

### JdbcRowSet

`JdbcRowSet` es un wrapper alrededor de un `ResultSet` convencional. Mantiene la conexion abierta y las operaciones se traducen directamente al ResultSet subyacente. No ofrece capacidad de desconexion. Su principal ventaja es que, al implementar la interfaz `RowSet`, puede usarse como JavaBean en entornos de diseño visual (IDE, herramientas de reportes).

```java
JdbcRowSet jrs = new JdbcRowSetImpl();
jrs.setUrl("jdbc:postgresql://localhost:5432/mi_bd");
jrs.setCommand("SELECT * FROM usuarios");
jrs.execute();

while (jrs.next()) {
    System.out.println(jrs.getString("nombre"));
}
```

A diferencia de usar directamente un `ResultSet`, `JdbcRowSet` permite registrar `RowSetListener` para recibir eventos de cambios de cursor, modificaciones de filas y cambios en el conjunto completo de resultados.

### JoinRowSet

`JoinRowSet` permite combinar datos de dos o mas RowSet en memoria, sin necesidad de una consulta SQL con JOIN. Es util cuando los datos provienen de distintas fuentes o cuando se quiere realizar una combinacion de datos en el cliente.

```java
CachedRowSet clientes = obtenerClientes(); // SELECT * FROM clientes
CachedRowSet pedidos = obtenerPedidos();   // SELECT * FROM pedidos

JoinRowSet jrs = new JoinRowSetImpl();
jrs.addRowSet(clientes, "id");      // Match column: clientes.id
jrs.addRowSet(pedidos, "cliente_id"); // Match column: pedidos.cliente_id

while (jrs.next()) {
    System.out.println(jrs.getString("nombre") + " - " + jrs.getInt("total"));
}
```

### FilteredRowSet

`FilteredRowSet` permite aplicar filtros sobre los datos en memoria sin volver a consultar la base de datos. El filtro se implementa mediante la interfaz `Predicate`.

```java
FilteredRowSet frs = new FilteredRowSetImpl();
frs.setUrl("jdbc:postgresql://localhost:5432/mi_bd");
frs.setCommand("SELECT * FROM productos");
frs.execute();

// Aplicar filtro en memoria (sin conexion)
frs.setFilter(new Predicate() {
    public boolean evaluate(RowSet rs) {
        try {
            return rs.getDouble("precio") > 100.0;
        } catch (SQLException e) {
            return false;
        }
    }
    // ... otros metodos evaluate
});

while (frs.next()) {
    System.out.println(frs.getString("nombre") + ": " + frs.getDouble("precio"));
}
```

## Uso en aplicaciones moviles

El caso de uso clasico de los RowSet desconectados es la aplicacion movil o el cliente offline. La aplicacion arranca, obtiene los datos necesarios mediante un `CachedRowSet` o `WebRowSet` (recibiendo incluso XML desde un servicio web), cierra la conexion y trabaja offline. Cuando vuelve a tener conectividad, aplica los cambios mediante `acceptChanges()` o serializa las modificaciones a XML para enviarlas al servidor.

```java
// Obtener datos al iniciar sesion
CachedRowSet catalogo = new CachedRowSetImpl();
catalogo.setUrl("jdbc:postgresql://servidor:5432/tienda");
catalogo.setCommand("SELECT * FROM productos WHERE activo = true");
catalogo.execute(); // Conexion -> datos -> desconexion

// Trabajar offline: el usuario navega, selecciona productos
// ...

// Sincronizar al reconectar
catalogo.acceptChanges();
```

> A pesar de su utilidad teorica, el uso de RowSet en aplicaciones modernas ha disminuido frente a alternativas como Spring Data REST, APIs RESTful con JSON, o soluciones de sincronizacion como Couchbase Lite o Realm. Los RowSet siguen siendo relevantes en entornos legacy, aplicaciones Java EE clasicas y herramientas de generacion de reportes donde la integracion con JavaBeans y el soporte de eventos son necesarios.
