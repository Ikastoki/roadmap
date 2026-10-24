# Vistas

Una vista es un objeto de base de datos que encapsula una consulta `SELECT` bajo un nombre, permitiendo referirse a ella como si fuera una tabla. A diferencia de una tabla, una vista no almacena datos fisicamente: cada vez que se accede a ella, PostgreSQL ejecuta la consulta subyacente y devuelve el resultado en tiempo real. Este mecanismo proporciona una capa de abstraccion que separa la complejidad del esquema subyacente de la interfaz que consumen las aplicaciones y los usuarios finales.

## Creacion y gestion de vistas

### CREATE VIEW

La sintaxis basica para crear una vista asigna un nombre a una consulta:

```sql
CREATE VIEW pedidos_resumen AS
SELECT c.nombre,
       COUNT(p.id) AS total_pedidos,
       SUM(p.total) AS importe_total
FROM clientes c
JOIN pedidos p ON p.cliente_id = c.id
GROUP BY c.id, c.nombre;
```

Una vez creada, `pedidos_resumen` se comporta como una tabla de solo lectura en la mayoria de los contextos:

```sql
SELECT * FROM pedidos_resumen WHERE importe_total > 1000;
```

La vista oculta la complejidad del JOIN y la agregacion. El planificador de PostgreSQL optimiza la consulta resultante igual que si se hubiera escrito el `SELECT` subyacente directamente, por lo que no hay penalizacion de rendimiento inherente al uso de vistas.

### CREATE OR REPLACE VIEW

Cuando una vista ya existe, `CREATE OR REPLACE VIEW` la redefine sin necesidad de eliminarla previamente. Esto preserva los permisos otorgados sobre la vista y evita ventanas de tiempo en las que la vista no esta disponible:

```sql
CREATE OR REPLACE VIEW pedidos_resumen AS
SELECT c.nombre,
       c.email,
       COUNT(p.id) AS total_pedidos,
       SUM(p.total) AS importe_total
FROM clientes c
JOIN pedidos p ON p.cliente_id = c.id
GROUP BY c.id, c.nombre, c.email;
```

No obstante, solo es posible reemplazar una vista si la nueva consulta genera las mismas columnas (mismos nombres y tipos) o si se anaden columnas al final. Cambiar el nombre, el tipo o el orden de columnas existentes requiere `DROP VIEW` seguido de un nuevo `CREATE VIEW`.

### DROP VIEW

Elimina una vista de la base de datos:

```sql
DROP VIEW IF EXISTS pedidos_resumen;
```

`IF EXISTS` evita un error si la vista no existe. La clausula `CASCADE` elimina tambien cualquier objeto que dependa de la vista (por ejemplo, otras vistas que la referencien):

```sql
DROP VIEW IF EXISTS pedidos_resumen CASCADE;
```

## Ventajas de las vistas

### Simplicidad

Las vistas encapsulan consultas complejas detras de un nombre simple. Un usuario o aplicacion puede consultar `ventas_mensuales` en lugar de escribir un `SELECT` con multiples JOIN, subconsultas y funciones de ventana cada vez. Esto reduce errores, centraliza la logica y facilita el mantenimiento: si la estructura de las tablas subyacentes cambia, solo es necesario actualizar la definicion de la vista, no todos los lugares que la usan.

### Seguridad

Mediante vistas es posible exponer solo un subconjunto de filas o columnas de una tabla, ocultando datos sensibles al usuario que consulta la vista. Por ejemplo, se puede crear una vista `empleados_publico` que omita las columnas `salario` y `numero_seguridad_social`, y otorgar permisos de lectura solo sobre la vista, no sobre la tabla base:

```sql
CREATE VIEW empleados_publico AS
SELECT id, nombre, departamento, email
FROM empleados;

GRANT SELECT ON empleados_publico TO rol_consulta;
```

El usuario con `rol_consulta` puede leer los datos expuestos por la vista pero no tiene acceso directo a la tabla `empleados`.

### Abstraccion

Las vistas desacoplan el esquema fisico de la interfaz logica. Si una tabla se normaliza, se particiona o se renombra, la vista puede ajustarse para que los consumidores externos no perciban el cambio. Esto es especialmente valioso en entornos donde multiples aplicaciones acceden a la misma base de datos y no es factible coordinarlas todas a la vez.

## Vistas actualizables

Una vista es actualizable cuando PostgreSQL puede traducir automaticamente operaciones `INSERT`, `UPDATE` y `DELETE` sobre la vista en operaciones equivalentes sobre las tablas base subyacentes. Para que una vista sea actualizable deben cumplirse todas las condiciones siguientes:

- La consulta `SELECT` de la vista debe tener una sola tabla base en su clausula `FROM` (sin JOIN, UNION, EXCEPT, INTERSECT ni subconsultas en FROM).
- La consulta no puede incluir `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`, `OFFSET`, `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` ni `FOR KEY SHARE`.
- La consulta no puede usar funciones de ventana, funciones de agregacion, ni operadores `SET` (UNION, INTERSECT, EXCEPT).
- Todas las columnas de la tabla base que sean `NOT NULL` y no tengan valor por defecto deben estar presentes en la vista.

Cuando se cumplen estas condiciones, un `INSERT` sobre la vista se convierte en un `INSERT` sobre la tabla base, un `UPDATE` modifica las filas correspondientes y un `DELETE` las elimina:

```sql
CREATE VIEW clientes_activos AS
SELECT id, nombre, email, activo
FROM clientes
WHERE activo = true;

-- Estas operaciones modifican directamente la tabla clientes:
INSERT INTO clientes_activos (nombre, email) VALUES ('Nuevo', 'nuevo@mail.com');
UPDATE clientes_activos SET email = 'cambiado@mail.com' WHERE id = 5;
DELETE FROM clientes_activos WHERE id = 3;
```

### WITH CHECK OPTION

Cuando una vista se utiliza para inserciones o actualizaciones, `WITH CHECK OPTION` garantiza que las filas modificadas sigan siendo visibles a traves de la vista. Sin esta clausula, es posible insertar una fila que la condicion `WHERE` de la vista excluiria, creando inconsistencias logicas:

```sql
CREATE VIEW clientes_activos AS
SELECT id, nombre, email, activo
FROM clientes
WHERE activo = true
WITH CHECK OPTION;

-- Esta insercion fallara porque activo = false viola la condicion WHERE de la vista
INSERT INTO clientes_activos (nombre, email, activo) VALUES ('Inactivo', 'i@mail.com', false);
-- ERROR: new row violates check option for view "clientes_activos"
```

`WITH CHECK OPTION` puede ser `LOCAL` (verifica solo las condiciones de la vista actual) o `CASCADED` (verifica las condiciones de la vista actual y de todas las vistas base que la componen). El valor por defecto es `CASCADED`.

> Aunque las vistas actualizables son practicas, no reemplazan las restricciones a nivel de tabla. `WITH CHECK OPTION` complementa pero no sustituye un `CHECK` constraint cuando la integridad debe garantizarse independientemente del camino de acceso a los datos.

