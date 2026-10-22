<h1 align="center">Vistas</h1>

<h2>📑 Contenido</h2>

- [Vistas](#vistas)
- [Creación de una vista](#creación-de-una-vista)
- [Modificación de una vista](#modificación-de-una-vista)
- [Eliminación de una vista](#eliminación-de-una-vista)

## Vistas

Las vistas en SQL son objetos virtuales que se basan en el conjunto de resultados de una consulta SQL. Funcionan como tablas virtuales y contienen filas y columnas, al igual que las tablas reales. Las vistas se crean para simplificar consultas complejas, personalizar la presentación de datos y mejorar la seguridad al limitar el acceso directo a las tablas subyacentes.

## Creación de una vista

Para crear una vista, utilizamos la sentencia CREATE VIEW.

```sql
CREATE VIEW Clientes_Brasil AS
SELECT CustomerName, ContactName
FROM Customers
WHERE Country = 'Brazil';
```

## Modificación de una vista

Para modificar una vista existente, utilizamos la sentencia CREATE OR REPLACE VIEW.

Esto nos permite redefinir la vista sin tener que eliminarla y volver a crearla.

```sql
CREATE OR REPLACE VIEW Clientes_Brasil AS
SELECT CustomerName, ContactName, City
FROM Customers
WHERE Country = 'Brazil';
```

## Eliminación de una vista

Para eliminar una vista, utilizamos la sentencia DROP VIEW

```sql
DROP VIEW Clientes_Brasil;
```
