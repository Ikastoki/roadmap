# DDL: creacion y manipulacion de tablas

Las tablas son la unidad fundamental de almacenamiento en una base de datos relacional. El DDL (_Data Definition Language_) proporciona las herramientas para definir, modificar y eliminar estas estructuras, ademas de los esquemas que las agrupan y los indices que optimizan su acceso. Cada operacion DDL en PostgreSQL es transaccional: si se ejecuta dentro de un bloque `BEGIN ... COMMIT` y ocurre un error, puede deshacerse completamente con `ROLLBACK`, una caracteristica que no todos los SGBD ofrecen.

## CREATE TABLE

La sentencia `CREATE TABLE` define una nueva tabla especificando sus columnas, tipos de datos y restricciones. La sintaxis general permite tanto restricciones inline (a nivel de columna) como restricciones de tabla (a nivel de tabla).

```sql
CREATE TABLE [IF NOT EXISTS] nombre_tabla (
    nombre_columna tipo_dato [restriccion_inline] [, ...]
    [, restriccion_tabla [, ...]]
) [INHERITS (tabla_padre)];
```

La clausula `IF NOT EXISTS` evita que la sentencia lance un error si la tabla ya existe, limitandose a mostrar un aviso. Es util en scripts de migracion que deben ser idempotentes.

### Columnas y restricciones inline

Cada columna se define con un nombre, un tipo de dato y opcionalmente una o mas restricciones:

```sql
CREATE TABLE empleados (
    id          INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nombre      VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    salario     NUMERIC(10,2) DEFAULT 0 CHECK (salario >= 0),
    activo      BOOLEAN DEFAULT TRUE,
    departamento_id INTEGER REFERENCES departamentos(id)
);
```

### Restricciones de tabla

Ciertas restricciones involucran multiples columnas y deben definirse como restricciones de tabla:

```sql
CREATE TABLE pedidos (
    id          INTEGER GENERATED ALWAYS AS IDENTITY,
    cliente_id  INTEGER NOT NULL,
    producto_id INTEGER NOT NULL,
    cantidad    INTEGER NOT NULL CHECK (cantidad > 0),
    fecha       DATE DEFAULT CURRENT_DATE,
    PRIMARY KEY (id),
    UNIQUE (cliente_id, producto_id, fecha),
    FOREIGN KEY (cliente_id) REFERENCES clientes(id) ON DELETE CASCADE,
    FOREIGN KEY (producto_id) REFERENCES productos(id) ON DELETE RESTRICT
);
```

## ALTER TABLE

`ALTER TABLE` permite modificar la estructura de una tabla existente sin necesidad de recrearla. Es una operacion potente que debe ejecutarse con cuidado en entornos productivos, pues algunas variantes (como cambiar el tipo de una columna) pueden requerir un bloqueo exclusivo de la tabla.

### ADD COLUMN

Anade una nueva columna a la tabla. Si se especifica `DEFAULT`, PostgreSQL lo aplica a todas las filas existentes en la misma operacion.

```sql
ALTER TABLE empleados ADD COLUMN telefono VARCHAR(20);
ALTER TABLE empleados ADD COLUMN fecha_ingreso DATE DEFAULT CURRENT_DATE;
```

### DROP COLUMN

Elimina una columna y todos sus datos. Si la columna es referenciada por una restriccion `FOREIGN KEY` u otro objeto, es necesario usar `CASCADE` para eliminar tambien las dependencias.

```sql
ALTER TABLE empleados DROP COLUMN telefono;
ALTER TABLE empleados DROP COLUMN telefono CASCADE;
```

### RENAME COLUMN y RENAME TO

Renombra columnas o la tabla misma sin modificar los datos almacenados.

```sql
ALTER TABLE empleados RENAME COLUMN telefono TO movil;
ALTER TABLE empleados RENAME TO trabajadores;
```

### ALTER COLUMN

Modifica el tipo de dato, el valor por defecto o la restriccion `NOT NULL` de una columna.

```sql
ALTER TABLE empleados ALTER COLUMN salario TYPE NUMERIC(12,2);
ALTER TABLE empleados ALTER COLUMN nombre SET NOT NULL;
ALTER TABLE empleados ALTER COLUMN nombre DROP NOT NULL;
ALTER TABLE empleados ALTER COLUMN salario SET DEFAULT 0;
ALTER TABLE empleados ALTER COLUMN salario DROP DEFAULT;
```

> El cambio de tipo de dato (`TYPE`) requiere una clausula `USING` cuando la conversion implicita no es suficiente. Por ejemplo, para convertir un texto en fecha: `ALTER TABLE pedidos ALTER COLUMN fecha TYPE DATE USING fecha::DATE;`

### ADD y DROP CONSTRAINT

Gestiona las restricciones de tabla por su nombre.

```sql
ALTER TABLE empleados ADD CONSTRAINT salario_positivo CHECK (salario >= 0);
ALTER TABLE empleados DROP CONSTRAINT salario_positivo;
```

## DROP TABLE

Elimina una tabla y todos sus datos de forma permanente. La clausula `IF EXISTS` evita errores si la tabla no existe.

```sql
DROP TABLE [IF EXISTS] nombre_tabla [CASCADE | RESTRICT];
```

`CASCADE` elimina automaticamente cualquier objeto que dependa de la tabla (vistas, foreign keys, funciones). `RESTRICT` (comportamiento por defecto) impide la eliminacion si existen dependencias.

```sql
DROP TABLE IF EXISTS empleados CASCADE;
```

## TRUNCATE

Elimina todas las filas de una tabla de forma mas eficiente que `DELETE` porque no genera registros de fila individual en el WAL y no verifica restricciones de foreign key durante la operacion. Sin embargo, no puede usarse si hay foreign keys que apunten a la tabla a menos que se incluya `CASCADE`.

```sql
TRUNCATE TABLE pedidos;
TRUNCATE TABLE pedidos RESTART IDENTITY;
TRUNCATE TABLE pedidos CASCADE;
TRUNCATE TABLE pedidos RESTART IDENTITY CASCADE;
```

La opcion `RESTART IDENTITY` reinicia las secuencias asociadas a columnas `IDENTITY` o `SERIAL`, haciendo que la proxima insercion comience desde el valor inicial. Es una practica habitual antes de recargar datos de prueba.

## CREATE INDEX

Los indices aceleran las consultas que filtran, ordenan o agrupan por las columnas indexadas. PostgreSQL soporta multiples tipos de indices, siendo B-Tree el predeterminado.

```sql
CREATE INDEX [CONCURRENTLY] [nombre_indice] ON tabla (columna1, columna2);
CREATE UNIQUE INDEX idx_empleados_email ON empleados (email);
CREATE INDEX idx_pedidos_fecha ON pedidos (fecha DESC);
```

La opcion `CONCURRENTLY` crea el indice sin bloquear operaciones concurrentes de escritura, aunque requiere mas tiempo y recursos.

### Indices parciales

Un indice puede limitarse a un subconjunto de filas mediante `WHERE`:

```sql
CREATE INDEX idx_pedidos_activos ON pedidos (fecha) WHERE estado = 'pendiente';
```

### Indices funcionales

Indexan el resultado de una expresion en lugar del valor de la columna:

```sql
CREATE INDEX idx_empleados_nombre_lower ON empleados (LOWER(nombre));
```

## CREATE SCHEMA

Los esquemas son contenedores logicos que agrupan tablas, vistas, funciones y otros objetos dentro de una misma base de datos. Permiten organizar el esquema en namespace separados y asignar permisos de forma granular.

```sql
CREATE SCHEMA [IF NOT EXISTS] nombre_esquema [AUTHORIZATION rol];
```

La clausula `AUTHORIZATION` asigna la propiedad del esquema a un rol especifico. Si no se especifica, el esquema pertenece al rol que ejecuta la sentencia.

```sql
CREATE SCHEMA ventas AUTHORIZATION gerente_ventas;
CREATE SCHEMA IF NOT EXISTS auditoria;
```

### search_path

El `search_path` define el orden en que PostgreSQL busca los objetos cuando no se califica su nombre con el esquema. Por defecto es `"$user", public`, lo que significa que primero busca en un esquema con el mismo nombre que el rol actual y despues en `public`.

```sql
SET search_path TO ventas, public;
SHOW search_path;
```

> Es posible crear varias tablas dentro de un esquema en una sola sentencia con la sintaxis `CREATE TABLE esquema.tabla (...)`. Si se omite el esquema, PostgreSQL usa el primero del `search_path`.

```sql
CREATE SCHEMA ventas;
CREATE TABLE ventas.clientes (id INTEGER PRIMARY KEY, nombre TEXT);
CREATE TABLE ventas.pedidos (id INTEGER PRIMARY KEY, cliente_id INTEGER);
```

