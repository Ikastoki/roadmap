<h1 align="center">Lenguaje de Definición (DDL)</h1>

<h2>📑 Contenido</h2>

- [Lenguaje de Definición (DDL)](#lenguaje-de-definición-ddl)
- [CREATE TABLE](#create-table)
- [SQL CREATE TABLE con NO NULL](#sql-create-table-con-no-null)
- [ALTER TABLE](#alter-table)
  - [Agregar una columna](#agregar-una-columna)
  - [Eliminar una columna](#eliminar-una-columna)
  - [Modificar una columna](#modificar-una-columna)
- [DROP TABLE](#drop-table)
- [TRUNCATE TABLE](#truncate-table)
- [CREATE INDEX](#create-index)
- [DROP INDEX](#drop-index)
- [CREATE VIEW](#create-view)
- [DROP VIEW](#drop-view)
- [Añadir/Drop Constraints](#añadirdrop-constraints)
  - [Añade limitaciones](#añade-limitaciones)
  - [Eliminar las restricciones](#eliminar-las-restricciones)
  - [Deja caer PRIMARY KEY](#deja-caer-primary-key)

## Lenguaje de Definición (DDL)

El lenguaje de definición de datos (DDL, por sus siglas en inglés) en SQL se utiliza para definir, modificar y eliminar la estructura de la base de datos y sus objetos relacionados.

## CREATE TABLE

Crea una nueva tabla en la base de datos.

```sql
CREATE TABLE nombre_de_la_tabla (
    columna1 tipo_de_dato,
    columna2 tipo_de_dato,
    ...
);

CREATE TABLE Employees (
    ID int,
    Name varchar(255),
    Salary int,
    Department varchar(255),
    Position varchar(255)
);
```

## SQL CREATE TABLE con NO NULL

El NOT NULLla restricción impone una columna para no aceptar valores nulos. Al crear una nueva tabla, puede añadir esta restricción

```sql
CREATE TABLE Employees (
    ID int NOT NULL,
    Name varchar(255) NOT NULL,
    Salary int,
    Department varchar(255),
    Position varchar(255)
);
```

## ALTER TABLE

Modifica la estructura de una tabla existente.

### Agregar una columna

```sql
ALTER TABLE nombre_de_la_tabla ADD COLUMN nombre_de_la_columna tipo_de_dato;
```

### Eliminar una columna

```sql
ALTER TABLE nombre_de_la_tabla DROP COLUMN nombre_de_la_columna;
```

### Modificar una columna

```sql
ALTER TABLE nombre_de_la_tabla ALTER COLUMN nombre_de_la_columna tipo_de_dato;
```

## DROP TABLE

Elimina una tabla de la base de datos.

```sql
DROP TABLE nombre_de_la_tabla;
```

## TRUNCATE TABLE

La instrucción TRUNCATE TABLE se utiliza en SQL para eliminar todos los registros de una tabla, pero sin eliminar la estructura de la tabla misma. Es una operación rápida y eficiente para borrar todos los datos de una tabla, ya que es más rápida que la instrucción DELETE, especialmente en tablas grandes, ya que no genera registros de deshacer y no activa disparadores de eliminación.

```sql
TRUNCATE TABLE nombre_de_la_tabla;
```

**Limitaciones**

- Se hace referencia a una restricción de FOREIGN KEY. (Puedes truncar una tabla que tiene una clave extranjera que se hace referencia a sí misma.)
- Participa en una opinión indexada.
- Se publica mediante la replicación transaccional o la replicación de la fusión.

## CREATE INDEX

Crea un índice en una tabla para acelerar las consultas.

```sql
CREATE INDEX nombre_del_indice ON nombre_de_la_tabla (columna);
```

## DROP INDEX

Elimina un índice de una tabla.

```sql
DROP INDEX nombre_del_indice;
```

## CREATE VIEW

Crea una vista, que es una consulta almacenada como un objeto.

```sql
CREATE VIEW nombre_de_la_vista AS
SELECT columna1, columna2 FROM nombre_de_la_tabla WHERE condición;
```

## DROP VIEW

Elimina una vista de la base de datos.

```sql
DROP VIEW nombre_de_la_vista;
```

## Añadir/Drop Constraints

### Añade limitaciones

```sql
ALTER TABLE tableName
ADD CONSTRAINT constraintName
PRIMARY KEY (column1, column2, ... column_n);
```

### Eliminar las restricciones

```sql
ALTER TABLE tableName
DROP CONSTRAINT constraintName;
```

### Deja caer PRIMARY KEY

```sql
ALTER TABLE table_name
DROP PRIMARY KEY;

In conclusion, `ALTER TABLE` in SQL lets you alter the structure of an existing table. This is a powerful command that lets you dynamically add, modify, and delete columns as well as the constraints placed on them. It ensures you are more flexible in dealing with changing data storage requirements.
```
