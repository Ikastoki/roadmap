# Row-Level Security

Los privilegios de tabla en PostgreSQL operan a nivel de objeto: si un rol tiene `SELECT` sobre una tabla, ve todas las filas. Para aplicaciones multiinquilino o sistemas con datos sensibles donde cada usuario solo debe ver sus propios registros, esta granularidad resulta insuficiente. Row-Level Security (RLS) resuelve este problema permitiendo definir politicas de acceso a nivel de fila directamente en el motor de base de datos.

## Habilitacion de RLS

RLS esta desactivado por defecto en todas las tablas. Se habilita explicitamente por tabla:

```sql
ALTER TABLE pedidos ENABLE ROW LEVEL SECURITY;
```

Una vez habilitado, todas las filas quedan invisibles a menos que exista una politica que conceda acceso al rol que realiza la consulta. El propietario de la tabla y los superusuarios siempre ven todas las filas, independientemente de las politicas.

Para forzar RLS incluso contra el propietario (recomendado en entornos multiinquilino):

```sql
ALTER TABLE pedidos FORCE ROW LEVEL SECURITY;
```

> `FORCE ROW LEVEL SECURITY` afecta a todos los roles excepto a superusuarios. Sin `FORCE`, el propietario de la tabla omite las politicas.

## CREATE POLICY

Una politica es una expresion SQL que se evalua para cada fila. Si la expresion devuelve `true`, la fila es visible o modificable segun el tipo de politica:

```sql
CREATE POLICY politica_cliente ON pedidos
    FOR SELECT
    USING (cliente_id = current_setting('app.cliente_id')::integer);
```

Las politicas se nombran dentro del esquema de la tabla. Los componentes basicos son:

- `FOR` especifica el comando: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, o `ALL` (todas las operaciones).
- `USING` determina que filas son visibles para comandos de lectura (`SELECT`, `UPDATE`, `DELETE`).
- `WITH CHECK` determina que filas pueden ser creadas o modificadas (`INSERT`, `UPDATE`).

### USING

La clausula `USING` filtra las filas existentes. Se evalua en el planificador como si fuera un `WHERE` adicional:

```sql
CREATE POLICY ver_pedidos ON pedidos FOR SELECT
    USING (cliente_id = current_setting('app.cliente_id')::integer);
```

Cuando un rol ejecuta `SELECT * FROM pedidos`, PostgreSQL anade automaticamente `AND cliente_id = current_setting('app.cliente_id')::integer` a la consulta. Si no se define `USING` para `SELECT`, el rol no ve ninguna fila.

### WITH CHECK

La clausula `WITH CHECK` se evalua sobre los valores que se intentan insertar o modificar. Si la expresion devuelve `false`, la operacion se rechaza con un error:

```sql
CREATE POLICY insertar_pedidos ON pedidos FOR INSERT
    WITH CHECK (cliente_id = current_setting('app.cliente_id')::integer);
```

Para `UPDATE`, ambas clausulas actuan de forma combinada: `USING` determina que filas existentes pueden ser modificadas, y `WITH CHECK` valida que los nuevos valores cumplan la politica. Esto impide que un usuario modifique una fila para que deje de cumplir la politica y quede inaccesible.

### ALL

`FOR ALL` aplica la politica a todos los comandos (`SELECT`, `INSERT`, `UPDATE`, `DELETE`):

```sql
CREATE POLICY admin_clientes ON clientes FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::integer)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::integer);
```

## Politicas por rol

Cada politica puede vincularse a uno o varios roles mediante `TO`. Si no se especifica `TO`, la politica aplica a todos los roles:

```sql
CREATE POLICY ver_propios ON pedidos FOR SELECT
    TO rol_cliente
    USING (cliente_id = current_setting('app.cliente_id')::integer);

CREATE POLICY ver_todos ON pedidos FOR SELECT
    TO rol_admin
    USING (true);
```

Cuando hay multiples politicas para el mismo comando y rol, se combinan con `OR`: si alguna politica concede acceso, la fila es accesible. Esto permite construir un modelo donde un rol hereda politicas de varios roles padre.

Para roles sin politica explicita, el comportamiento depende de si se ha definido alguna politica para ese comando. Si existe al menos una politica para `SELECT` en la tabla pero ninguna aplica al rol, el rol no ve filas. Si no existe ninguna politica para `SELECT`, RLS no filtra nada (aunque este habilitado).

## Ejemplo completo: multiinquilino

Un escenario tipico de RLS es una aplicacion SaaS donde cada cliente (inquilino) solo ve sus datos:

```sql
-- esquema y tabla
CREATE SCHEMA saas;
CREATE TABLE saas.facturas (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    importe NUMERIC(10,2),
    fecha DATE
);

-- habilitar RLS
ALTER TABLE saas.facturas ENABLE ROW LEVEL SECURITY;
ALTER TABLE saas.facturas FORCE ROW LEVEL SECURITY;

-- politica
CREATE POLICY facturas_tenant ON saas.facturas FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::integer)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::integer);

-- roles
CREATE ROLE tenant_1 WITH LOGIN PASSWORD 'pass1';
CREATE ROLE tenant_2 WITH LOGIN PASSWORD 'pass2';

-- la aplicacion establece el identificador del inquilino al conectar
-- SET app.tenant_id = '42';
```

La aplicacion configura `app.tenant_id` al iniciar la sesion (tipicamente mediante `SET` en el pool de conexiones o en la configuracion del driver). Todas las consultas posteriores heredan el filtro automaticamente, sin que la aplicacion necesite incluir `WHERE tenant_id = ?` en cada sentencia.

## Consideraciones de rendimiento

RLS anade condiciones a todas las consultas sobre la tabla, lo que puede afectar al rendimiento si las politicas contienen subconsultas complejas. Es recomendable:

- Usar columnas indexadas en las expresiones de `USING` y `WITH CHECK`.
- Evitar llamadas a funciones costosas dentro de las politicas.
- Preferir `current_setting()` a subconsultas, ya que es una lectura en memoria sin acceso a disco.
- Verificar el plan de ejecucion con `EXPLAIN` para confirmar que los indices se utilizan.

> Las politicas se evaluan por fila. En tablas grandes, una politica ineficiente puede degradar drasticamente el rendimiento de operaciones que normalmente serian scans secuenciales.

