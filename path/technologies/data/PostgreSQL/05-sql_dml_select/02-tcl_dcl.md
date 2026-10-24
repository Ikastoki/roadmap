# TCL y DCL: transacciones, permisos y roles

Las sentencias DML modifican datos, pero sin un control transaccional cualquier operacion es potencialmente peligrosa: un error a mitad de una actualizacion masiva puede dejar los datos en un estado inconsistente. TCL (_Transaction Control Language_) proporciona los mecanismos para delimitar, confirmar o deshacer unidades de trabajo. DCL (_Data Control Language_), por su parte, regula quien puede hacer que, estableciendo el modelo de seguridad del acceso a los datos.

## TCL

### BEGIN, COMMIT y ROLLBACK

En PostgreSQL, toda operacion se ejecuta dentro de una transaccion. El modo autocommit (predeterminado en psql y la mayoria de clientes) ejecuta cada sentencia como su propia transaccion. Para agrupar varias operaciones, se delimitan explicitamente con `BEGIN` y `COMMIT`:

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;
```

Si algo falla o se decide abortar, `ROLLBACK` deshace todos los cambios desde el `BEGIN`:

```sql
BEGIN;
DELETE FROM pedidos WHERE cliente_id = 999;
-- algo salio mal
ROLLBACK;
```

> PostgreSQL garantiza la durabilidad de las transacciones confirmadas mediante WAL. Si el servidor se cae justo despues de `COMMIT`, los datos se recuperan del WAL en el arranque.

### Autocommit

El autocommit no es una construccion del lenguaje SQL sino un comportamiento del cliente. En psql, esta activo por defecto: cada sentencia se confirma inmediatamente. Se puede desactivar con `\set AUTOCOMMIT off` o explicitamente con `BEGIN`. En librerias como psycopg2 (Python) o libpq, el autocommit se controla mediante la configuracion de la conexion.

### SAVEPOINT, ROLLBACK TO y RELEASE

Los savepoints permiten establecer puntos de restauracion dentro de una transaccion, de modo que se pueda deshacer parte del trabajo sin abortar la transaccion completa:

```sql
BEGIN;
INSERT INTO logs (evento) VALUES ('inicio_proceso');
SAVEPOINT punto_control;
UPDATE productos SET stock = stock - 1 WHERE id = 10;
-- detectamos que no debimos descontar
ROLLBACK TO punto_control;
-- el INSERT anterior permanece, el UPDATE se deshizo
INSERT INTO logs (evento) VALUES ('error_corregido');
COMMIT;
```

`ROLLBACK TO savepoint` deshace todo lo ocurrido despues del savepoint pero mantiene el savepoint activo. Para eliminarlo sin deshacer cambios se usa `RELEASE SAVEPOINT`:

```sql
BEGIN;
SAVEPOINT sp1;
INSERT INTO tabla VALUES (1);
RELEASE SAVEPOINT sp1;
-- sp1 ya no existe, pero el INSERT permanece
COMMIT;
```

Los savepoints son especialmente utiles en bucles dentro de funciones PL/pgSQL, donde se puede capturar una excepcion, deshacer el trabajo del paso actual y continuar con el siguiente sin abortar toda la transaccion.

### ISOLATION LEVEL

PostgreSQL soporta cuatro niveles de aislamiento definidos en el estandar SQL, aunque internamente implementa tres comportamientos distintos:

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

`READ UNCOMMITTED` se acepta sintacticamente pero se trata como `READ COMMITTED`. La tabla comparativa:

| Nivel | Lectura sucia | Lectura no repetible | Lectura fantasma |
|---|---|---|---|
| READ COMMITTED | No | Si | Si |
| REPEATABLE READ | No | No | Si |
| SERIALIZABLE | No | No | No |

`READ COMMITTED` es el nivel por defecto. Cada sentencia dentro de la transaccion ve los datos confirmados hasta el momento de esa consulta. `REPEATABLE READ` congela una snapshot al inicio de la primera consulta, de modo que todas las lecturas posteriores ven los mismos datos. `SERIALIZABLE` utiliza tecnicas de deteccion de conflictos basadas en SIREAD locks para garantizar que la ejecucion concurrente sea equivalente a algun orden serial.

### READ ONLY y READ WRITE

Una transaccion puede marcarse como readonly, lo que permite a PostgreSQL optimizar internamente (por ejemplo, evitando la asignacion de XID):

```sql
BEGIN;
SET TRANSACTION READ ONLY;
SELECT * FROM reportes WHERE fecha > '2025-01-01';
COMMIT;
```

Cualquier intento de `INSERT`, `UPDATE` o `DELETE` dentro de una transaccion readonly produce un error. Esta caracteristica es util para asegurar que consultas de solo lectura no modifiquen accidentalmente datos.

## DCL

### GRANT

`GRANT` otorga privilegios sobre objetos de la base de datos a uno o varios roles. Los privilegios se organizan jerarquicamente desde el nivel de conexion hasta el nivel de columna:

```sql
-- privilegios de conexion
GRANT CONNECT ON DATABASE mi_bd TO rol_lectura;

-- privilegios sobre esquemas (necesario para acceder a objetos dentro)
GRANT USAGE ON SCHEMA ventas TO rol_lectura;

-- privilegios sobre tablas
GRANT SELECT ON ventas.productos TO rol_lectura;
GRANT INSERT, UPDATE, DELETE ON ventas.pedidos TO rol_escritura;

-- privilegios sobre columnas especificas
GRANT UPDATE (nombre, email) ON clientes TO rol_actualizador;

-- privilegios sobre funciones
GRANT EXECUTE ON FUNCTION calcular_impuestos(integer) TO rol_contabilidad;

-- todos los privilegios
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA ventas TO rol_admin;
```

> `USAGE` en un esquema no otorga ningun permiso sobre los objetos internos; solo permite "ver" el esquema y usar los objetos sobre los que se tengan permisos especificos. Sin `USAGE`, una tabla visible no es accesible aunque se tenga `SELECT` sobre ella.

### REVOKE

`REVOKE` retira privilegios previamente otorgados. Soporta las mismas variantes que `GRANT`:

```sql
REVOKE INSERT ON pedidos FROM rol_escritura;
REVOKE ALL PRIVILEGES ON SCHEMA ventas FROM rol_lectura;
REVOKE CONNECT ON DATABASE mi_bd FROM PUBLIC;
```

La palabra clave `PUBLIC` representa a todos los roles, incluidos los que se creen en el futuro. Revocar privilegios de PUBLIC es una practica de seguridad basica.

### Roles

Los roles en PostgreSQL unifican los conceptos de usuario y grupo. Un rol puede iniciar sesion (si tiene `LOGIN`), pertenecer a otros roles y heredar sus privilegios.

#### CREATE ROLE

```sql
CREATE ROLE analista WITH LOGIN PASSWORD 'secreto123';
CREATE ROLE desarrollador WITH LOGIN PASSWORD 'dev_pass' VALID UNTIL '2026-12-31';
CREATE ROLE solo_lectura;
CREATE ROLE adm_bd WITH LOGIN SUPERUSER CREATEDB CREATEROLE;
```

Los atributos principales de un rol son:

- `LOGIN`: permite conectar con el rol (equivalente a CREATE USER, que es un alias de CREATE ROLE WITH LOGIN).
- `PASSWORD`: establece la contrasena de autenticacion.
- `SUPERUSER`: sin restricciones de seguridad. Solo un superuser puede crear otros superusers.
- `CREATEDB`: permite crear bases de datos.
- `CREATEROLE`: permite crear y gestionar otros roles.
- `INHERIT`: hereda los privilegios de los roles a los que pertenece (comportamiento por defecto).
- `VALID UNTIL`: fecha de expiracion del rol.

#### GRANT rol a rol

Los roles se organizan jerarquicamente otorgando un rol a otro:

```sql
CREATE ROLE ventas_lectura;
CREATE ROLE ventas_escritura;
CREATE ROLE ana WITH LOGIN PASSWORD 'ana123';

GRANT SELECT ON ALL TABLES IN SCHEMA ventas TO ventas_lectura;
GRANT INSERT, UPDATE ON ALL TABLES IN SCHEMA ventas TO ventas_escritura;

GRANT ventas_lectura TO ventas_escritura;
GRANT ventas_escritura TO ana;
```

Ana hereda todos los privilegios de `ventas_escritura`, que a su vez hereda los de `ventas_lectura`. La herencia es transitiva.

#### ALTER ROLE

Modifica atributos de un rol existente:

```sql
ALTER ROLE analista SET search_path TO ventas, public;
ALTER ROLE desarrollador WITH NOSUPERUSER;
ALTER ROLE ana RENAME TO ana_nueva;
ALTER ROLE analista VALID UNTIL '2027-06-01';
```

#### SET search_path para roles

Es posible fijar un `search_path` especifico para un rol, de modo que al conectar tenga visibilidad directa de ciertos esquemas sin necesidad de calificar los objetos:

```sql
ALTER ROLE ventas_usuarios SET search_path TO ventas, public;
```

Este cambio solo afecta a nuevas conexiones del rol. Dentro de una sesion se puede modificar con `SET search_path TO ...`.

