# Restricciones, relaciones y esquemas

La integridad de los datos en una base de datos relacional se garantiza mediante restricciones que el motor evalua automaticamente en cada operacion de escritura. PostgreSQL implementa el estandar SQL de forma rigurosa, incluyendo restricciones `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `EXCLUDE` y la capacidad de diferir su verificacion al final de la transaccion. Ademas, el sistema de esquemas permite organizar los objetos logicamente y controlar los permisos de acceso a nivel de namespace.

## Restricciones de tabla

Las restricciones (_constraints_) son reglas que el motor de base de datos impone sobre los datos para preservar la integridad. Se definen en el `CREATE TABLE` o se anaden posteriormente con `ALTER TABLE`.

### NOT NULL

Garantiza que una columna no contenga valores nulos. Es la restriccion mas basica y se define unicamente a nivel de columna.

```sql
CREATE TABLE productos (
    nombre    TEXT NOT NULL,
    precio    NUMERIC(10,2) NOT NULL
);
ALTER TABLE productos ALTER COLUMN nombre SET NOT NULL;
ALTER TABLE productos ALTER COLUMN nombre DROP NOT NULL;
```

### DEFAULT

No es una restriccion en sentido estricto, sino un valor que se asigna automaticamente cuando la columna se omite en un `INSERT`.

```sql
CREATE TABLE pedidos (
    fecha       DATE DEFAULT CURRENT_DATE,
    estado      TEXT DEFAULT 'pendiente',
    total       NUMERIC(10,2) DEFAULT 0
);
```

### UNIQUE

Asegura que todos los valores de una columna o combinacion de columnas sean distintos. PostgreSQL crea automaticamente un indice B-Tree unico para cada restriccion `UNIQUE`.

```sql
CREATE TABLE usuarios (
    email   TEXT UNIQUE,
    alias   TEXT,
    UNIQUE (alias)
);
-- UNIQUE compuesto:
CREATE TABLE inscripciones (
    curso_id    INTEGER,
    alumno_id   INTEGER,
    UNIQUE (curso_id, alumno_id)
);
```

### PRIMARY KEY

Combina `NOT NULL` y `UNIQUE` en una sola restriccion. Cada tabla puede tener una unica clave primaria, aunque esta puede ser compuesta (multiples columnas). PostgreSQL crea un indice B-Tree unico para la PK.

```sql
CREATE TABLE clientes (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nif TEXT UNIQUE NOT NULL
);
-- Clave primaria compuesta:
CREATE TABLE matriculas (
    alumno_id   INTEGER,
    curso_id    INTEGER,
    fecha_alta  DATE,
    PRIMARY KEY (alumno_id, curso_id)
);
```

### FOREIGN KEY

Garantiza la integridad referencial: los valores de la columna deben existir en la columna referenciada de otra tabla. Puede definirse inline o como restriccion de tabla.

```sql
CREATE TABLE pedidos (
    id          INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    cliente_id  INTEGER REFERENCES clientes(id),
    producto_id INTEGER,
    FOREIGN KEY (producto_id) REFERENCES productos(id)
);
```

### CHECK

Valida que una expresion booleana sea verdadera (o desconocida) para cada fila. Si la expresion evalua a `FALSE`, la operacion se rechaza.

```sql
CREATE TABLE empleados (
    salario     NUMERIC(10,2) CHECK (salario >= 0),
    fecha_nac   DATE CHECK (fecha_nac < CURRENT_DATE),
    CHECK (salario > 0 OR comision IS NOT NULL)
);
```

## Acciones referenciales: ON DELETE y ON UPDATE

Cuando una fila referenciada por una `FOREIGN KEY` se elimina o actualiza, el comportamiento se define mediante las clausulas `ON DELETE` y `ON UPDATE`. La eleccion de la accion depende de la semantica del negocio.

```sql
CREATE TABLE pedidos (
    id          INTEGER PRIMARY KEY,
    cliente_id  INTEGER REFERENCES clientes(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    producto_id INTEGER REFERENCES productos(id)
        ON DELETE RESTRICT
        ON UPDATE SET NULL
);
```

| Accion | Descripcion |
|--------|-------------|
| `CASCADE` | Propaga la operacion: elimina o actualiza las filas dependientes |
| `SET NULL` | Establece a `NULL` las columnas de la foreign key |
| `SET DEFAULT` | Asigna el valor por defecto de la columna |
| `RESTRICT` | Impide la operacion si hay filas dependientes (se evalua inmediatamente) |
| `NO ACTION` | Similar a `RESTRICT`, pero la verificacion se difiere si la constraint es deferrable (por defecto se comporta como `RESTRICT`) |

> `RESTRICT` y `NO ACTION` son funcionalmente identicos en PostgreSQL a menos que la constraint sea `DEFERRABLE`. Con `NO ACTION` y `DEFERRABLE`, la verificacion se pospone hasta el final de la transaccion, permitiendo operaciones que temporalmente violen la integridad referencial.

## Exclusion constraints

Las restricciones de exclusion garantizan que no existan dos filas que cumplan una condicion de conflicto. Son especialmente utiles para solapamiento de rangos, reservas de recursos y conflictos geometricos.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE reservas_salas (
    sala    INTEGER,
    desde   TIMESTAMP,
    hasta   TIMESTAMP,
    EXCLUDE USING gist (
        sala WITH =,
        tsrange(desde, hasta) WITH &&
    )
);
```

Esta restriccion impide insertar dos reservas para la misma sala cuyos periodos de tiempo se solapen (`&&` operador de solapamiento para rangos).

## Deferrable constraints

Por defecto, las restricciones se evaluan inmediatamente despues de cada sentencia (`NOT DEFERRABLE`). Sin embargo, es posible declarar una constraint como `DEFERRABLE` para que su verificacion se posponga hasta el final de la transaccion, lo que permite violaciones temporales durante la ejecucion de varias sentencias.

```sql
CREATE TABLE cuentas (
    id      INTEGER PRIMARY KEY,
    saldo   NUMERIC(10,2) CHECK (saldo >= 0) DEFERRABLE INITIALLY DEFERRED
);
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;  -- saldo temporalmente negativo
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;  -- se compensa
COMMIT;  -- la restriccion se verifica aqui: si todo cuadra, se confirma
```

```sql
-- Diferir una constraint existente solo para la transaccion actual:
SET CONSTRAINTS fk_pedidos_cliente DEFERRED;
```

## SERIAL vs GENERATED AS IDENTITY

Las columnas de tipo `SERIAL` son un atajo pre-estandar que crea implicitamente una secuencia. Desde PostgreSQL 10, la sintaxis `GENERATED AS IDENTITY` (definida en SQL:2003) es la opcion recomendada.

```sql
-- Forma clasica (SERIAL)
CREATE TABLE tabla_serial (
    id SERIAL PRIMARY KEY
);

-- Forma estandar (IDENTITY) - recomendada
CREATE TABLE tabla_identity (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY
    -- o GENERATED BY DEFAULT AS IDENTITY
);
```

Diferencias clave:

| Aspecto | SERIAL | GENERATED AS IDENTITY |
|---------|--------|-----------------------|
| Estandar | Propietario (pre-SQL:2003) | SQL:2003 |
| Control de insercion | Se puede insertar cualquier valor manualmente | `GENERATED ALWAYS` impide insercion manual |
| Propiedad de secuencia | La secuencia es independiente | La secuencia pertenece a la columna |
| Alteracion | Depende de `ALTER SEQUENCE` | `ALTER TABLE ... ALTER COLUMN ... RESTART` |
| Seguridad | Puede sobrescribirse facilmente | Previene sobrescritura accidental |

```sql
-- Con GENERATED ALWAYS, esto falla:
INSERT INTO tabla_identity (id) VALUES (100);  -- ERROR

-- Con GENERATED BY DEFAULT, permite valor explicito:
INSERT INTO tabla_identity (id) VALUES (100);  -- OK

-- Para resetear la identidad:
ALTER TABLE tabla_identity ALTER COLUMN id RESTART WITH 1;
```

## Relaciones entre tablas

Las relaciones modelan las asociaciones entre entidades del dominio. Se implementan mediante claves foraneas que referencian claves primarias (o columnas unicas) de otras tablas.

### One-to-Many (1:N)

La relacion mas comun. Una fila en la tabla A se asocia con varias filas en la tabla B.

```sql
CREATE TABLE departamentos (
    id   INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nombre TEXT NOT NULL
);
CREATE TABLE empleados (
    id              INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nombre          TEXT NOT NULL,
    departamento_id INTEGER NOT NULL REFERENCES departamentos(id)
);
CREATE INDEX idx_empleados_departamento ON empleados (departamento_id);
```

### Many-to-Many (N:M)

Requiere una tabla intermedia (_junction table_) que contiene las claves foraneas de ambas tablas. La clave primaria suele ser compuesta.

```sql
CREATE TABLE estudiantes (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nombre TEXT NOT NULL
);
CREATE TABLE cursos (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    titulo TEXT NOT NULL
);
CREATE TABLE inscripciones (
    estudiante_id INTEGER REFERENCES estudiantes(id) ON DELETE CASCADE,
    curso_id      INTEGER REFERENCES cursos(id) ON DELETE CASCADE,
    fecha         DATE DEFAULT CURRENT_DATE,
    PRIMARY KEY (estudiante_id, curso_id)
);
CREATE INDEX idx_inscripciones_curso ON inscripciones (curso_id);
```

### One-to-One (1:1)

Se implementa con una foreign key que ademas tiene una restriccion `UNIQUE`, asegurando que cada fila de la tabla A se asocie con, como maximo, una fila de la tabla B.

```sql
CREATE TABLE usuarios (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    username TEXT UNIQUE NOT NULL
);
CREATE TABLE perfiles (
    usuario_id INTEGER PRIMARY KEY REFERENCES usuarios(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url TEXT
);
```

## Esquemas

Un esquema es un namespace que agrupa objetos de base de datos (tablas, vistas, funciones, tipos). Cada base de datos contiene al menos el esquema `public`, que es el utilizado por defecto.

### El esquema public

Al crear una base de datos, PostgreSQL crea automaticamente el esquema `public`. Todos los objetos creados sin calificar con un esquema se almacenan en el primer esquema del `search_path` (tipicamente `public`).

```sql
SELECT current_schema();  -- muestra el esquema activo
SHOW search_path;         -- muestra el orden de busqueda
```

### Crear y usar esquemas

Los esquemas se crean con `CREATE SCHEMA` y se referencian anteponiendo el nombre al objeto:

```sql
CREATE SCHEMA ventas;
CREATE SCHEMA inventario;

CREATE TABLE ventas.clientes (id INTEGER PRIMARY KEY, nombre TEXT);
CREATE TABLE inventario.productos (id INTEGER PRIMARY KEY, nombre TEXT, stock INTEGER);

-- Consulta calificada:
SELECT * FROM ventas.clientes;
```

### Permisos por esquema

Los esquemas permiten controlar el acceso de forma granular. Para que un rol pueda ver los objetos de un esquema, necesita el privilegio `USAGE` sobre el esquema.

```sql
CREATE SCHEMA recursos_humano;
GRANT USAGE ON SCHEMA recursos_humano TO gerente_rrhh;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA recursos_humano TO gerente_rrhh;
ALTER DEFAULT PRIVILEGES IN SCHEMA recursos_humano
    GRANT SELECT ON TABLES TO analista_rrhh;
```

`ALTER DEFAULT PRIVILEGES` asegura que las tablas creadas en el futuro en ese esquema hereden los permisos especificados, eliminando la necesidad de concederlos manualmente tras cada `CREATE TABLE`.

