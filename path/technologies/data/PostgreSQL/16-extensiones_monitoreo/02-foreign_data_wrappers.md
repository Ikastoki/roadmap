# Foreign Data Wrappers

El modelo SQL/MED (Management of External Data) del estandar SQL:2003 define un marco para que una base de datos acceda a datos almacenados fuera de ella como si fueran tablas locales. PostgreSQL implementa este modelo mediante el mecanismo de Foreign Data Wrappers (FDW), que permite conectar y consultar fuentes de datos heterogeneas --otras bases de datos, archivos planos, APIs web, sistemas NoSQL-- sin mover los datos ni depender de procesos ETL externos. Un FDW no es solo un puente de conectividad: es un adaptador que traduce el lenguaje de consulta de PostgreSQL al protocolo nativo del sistema remoto, delegando parte del procesamiento cuando es posible.

## Que es un FDW

Un Foreign Data Wrapper es una extension que define un conjunto de funciones callback para conectar, ejecutar consultas y retornar resultados desde un origen de datos externo. Cada FDW implementa el protocolo especifico del sistema remoto. PostgreSQL provee FDW oficiales para varias fuentes: `postgres_fdw` (PostgreSQL remoto), `mysql_fdw`, `oracle_fdw`, `file_fdw` (archivos planos), `redis_fdw`, entre otros.

La arquitectura consta de varios componentes:

- **Foreign Data Wrapper:** la libreria que implementa el protocolo de acceso.
- **Foreign Server:** definicion que agrupa los parametros de conexion (host, puerto, base de datos).
- **User Mapping:** mapeo de credenciales entre el usuario local y el remoto.
- **Foreign Table:** tabla virtual que refleja la estructura de la tabla remota.
- **Import Foreign Schema:** comando para importar automaticamente la estructura de multiples tablas.

## Ciclo completo con postgres_fdw

### 1. Crear la extension

```sql
CREATE EXTENSION postgres_fdw;
```

### 2. Crear el servidor remoto

`CREATE SERVER` define un servidor extranjero asociado a un FDW concreto:

```sql
CREATE SERVER servidor_remoto
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (
    host '192.168.1.100',
    port '5432',
    dbname 'bd_remota'
);
```

Las opciones dependen del FDW. `postgres_fdw` acepta `host`, `port`, `dbname`, `connect_timeout`, `sslmode`, `sslrootcert`, entre otras.

### 3. Crear el mapeo de usuario

`CREATE USER MAPPING` asocia un rol local con las credenciales del servidor remoto. Las contrasenas se almacenan en el catalogo del sistema en texto plano, por lo que el acceso a `pg_user_mappings` debe restringirse:

```sql
CREATE USER MAPPING FOR usuario_local
SERVER servidor_remoto
OPTIONS (user 'user_remoto', password 'secreto');
```

El mapeo puede crearse para un rol especifico o para `PUBLIC`, aunque esto ultimo no es recomendable.

### 4. Crear tablas extranjeras

`CREATE FOREIGN TABLE` define una tabla virtual cuya estructura debe coincidir (en columnas y tipos compatibles) con la tabla remota:

```sql
CREATE FOREIGN TABLE ventas_remotas (
    id INTEGER,
    producto TEXT,
    cantidad INTEGER,
    precio NUMERIC(10,2),
    fecha DATE
)
SERVER servidor_remoto
OPTIONS (
    schema_name 'public',
    table_name 'ventas'
);
```

Una vez creada, se consulta como cualquier tabla local:

```sql
SELECT producto, SUM(cantidad) AS total
FROM ventas_remotas
WHERE fecha >= '2025-01-01'
GROUP BY producto;
```

### 5. Importar esquemas remotos

Para evitar crear cada tabla manualmente, `IMPORT FOREIGN SCHEMA` genera automaticamente tablas extranjeras a partir de las tablas del schema remoto:

```sql
IMPORT FOREIGN SCHEMA public
FROM SERVER servidor_remoto
INTO schema_local;

-- Limitado a tablas que coincidan con un patron
IMPORT FOREIGN SCHEMA public
LIMIT TO (clientes, ventas)
FROM SERVER servidor_remoto
INTO schema_local;

-- Excluyendo tablas
IMPORT FOREIGN SCHEMA public
EXCEPT (logs_temporales)
FROM SERVER servidor_remoto
INTO schema_local;
```

## Push-down de operaciones

La capacidad de delegar procesamiento al servidor remoto se conoce como _push-down_ y es el factor que determina el rendimiento de las consultas FDW. `postgres_fdw` puede enviar al remoto las siguientes operaciones:

- **Filtros WHERE:** las condiciones en la consulta se traducen al SQL remoto.
- **Columnas SELECT:** solo se solicitan las columnas necesarias.
- **JOINs:** desde PostgreSQL 12, los JOINs entre tablas del mismo servidor remoto pueden ejecutarse alli.
- **ORDER BY y LIMIT:** la ordenacion y limite se delegan cuando es seguro.
- **Funciones y operadores:** siempre que sean compatibles entre ambos sistemas.

```sql
-- El planificador decide que enviar al remoto
EXPLAIN (VERBOSE)
SELECT p.nombre, v.total
FROM ventas_remotas v
JOIN productos_remotos p ON v.producto_id = p.id
WHERE v.fecha >= '2025-06-01'
ORDER BY v.total DESC
LIMIT 10;
```

En el plan de ejecucion, las operaciones que se ejecutan en el remoto aparecen como `Foreign Scan` y las de procesamiento local como `Sort`, `Hash Join`, etc.

> No todos los FDW implementan push-down con la misma profundidad. `postgres_fdw` es de los mas completos, mientras que `file_fdw` no puede delegar nada porque la fuente es un archivo plano sin capacidad de procesamiento.

## Consideraciones de rendimiento

Aunque los FDW abstraen la complejidad de la fuente externa, introducen latencia de red y overhead de serializacion. Algunas practicas recomendadas:

- **Fetch size:** configurar el numero de filas que se recuperan por viaje de ida y vuelta:

```sql
ALTER SERVER servidor_remoto OPTIONS (ADD fetch_size '500');
```

- **Uso de estadisticas:** ANALYZE sobre tablas extranjeras permite al planificador estimar cardinalidades:

```sql
ANALYZE ventas_remotas;
```

- **Conexiones persistentes:** `postgres_fdw` reutiliza conexiones dentro de una sesion, pero puede configurarse un pool con `pool_size` y `connection_life_time`.

- **Escritura remota:** admiten `INSERT`, `UPDATE`, `DELETE` e incluso `TRUNCATE` desde PostgreSQL 11. Sin embargo, cada operacion DML genera una transaccion remota independiente, lo que afecta al rendimiento en operaciones masivas.

## FDW frente a otras alternativas

| Aspecto | FDW | ETL | Replicacion logica |
|---------|-----|-----|--------------------|
| Latencia | Consulta directa | Datos pre-cargados | Replica continua |
| Actualizacion de datos | Tiempo real | Depende del ciclo ETL | Tiempo real |
| Carga en origen | Si (consulta remota) | No (copia previa) | Si (log de cambios) |
| Independencia de schema | Alta | Media | Baja (debe coincidir) |
| Uso tipico | Federacion de datos, integracion | Data warehouse, migraciones | Alta disponibilidad, escalado lectura |

La eleccion entre FDW, replicacion y ETL depende del caso de uso: los FDW brillan en escenarios de integracion ligera y consultas ocasionales a sistemas externos, pero una replicacion dedicada ofrece mejor rendimiento para consultas frecuentes sobre grandes volumenes.

