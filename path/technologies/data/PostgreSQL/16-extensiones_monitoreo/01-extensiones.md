# Extensiones populares y su gestion

PostgreSQL se disena desde sus origenes como un sistema extensible. Su arquitectura permite cargar modulos que anaden funcionalidades sin modificar el nucleo del motor, lo que lo diferencia de otros RDBMS donde las capacidades avanzadas suelen estar integradas de forma monolitica o requieren forks del codigo fuente. Este mecanismo de extensiones convierte a PostgreSQL en una plataforma que se adapta a dominios tan diversos como la busqueda textual, la geolocalizacion, la criptografia o la administracion de particiones, todo sin comprometer la estabilidad del nucleo.

## Gestion de extensiones

### CREATE EXTENSION

El comando `CREATE EXTENSION` instala y activa una extension en una base de datos concreta. Internamente, ejecuta un script SQL que crea las funciones, tipos, operadores y tablas que la componen.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

La clausula `IF NOT EXISTS` evita errores si la extension ya esta instalada. Algunas extensiones permiten especificar un schema de destino:

```sql
CREATE EXTENSION postgis SCHEMA public;
```

### DROP EXTENSION

Para eliminar una extension y todos sus objetos asociados:

```sql
DROP EXTENSION IF EXISTS pg_trgm CASCADE;
```

La opcion `CASCADE` elimina objetos que dependan de la extension. Sin ella, el comando falla si existen dependencias.

### pg_extension

La tabla del catalogo `pg_extension` almacena las extensiones instaladas en la base de datos actual:

```sql
SELECT extname, extversion, extowner, extnamespace
FROM pg_extension;
```

Cada fila contiene el nombre de la extension, su version, el propietario (rol) y el schema donde reside.

### pg_available_extensions

La vista `pg_available_extensions` lista todas las extensiones disponibles en el sistema, hayan sido instaladas o no:

```sql
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions;
```

Esta vista es util para descubrir que extensiones pueden instalarse sin necesidad de consultar la documentacion. La columna `installed_version` es `NULL` si la extension no esta instalada.

## Extensiones populares

### pg_trgm (similitud de cadenas)

La extension `pg_trgm` proporciona funciones y operadores para determinar la similitud entre cadenas basandose en trigramas (secuencias de tres caracteres consecutivos). Es la base para busqueda aproximada (_fuzzy search_) y autocompletado.

```sql
CREATE EXTENSION pg_trgm;

-- Similitud entre dos cadenas (0 a 1)
SELECT similarity('programacion', 'programador');  -- 0.727...

-- Operador de similitud
SELECT 'programacion' % 'programador';  -- true si supera el umbral

-- Distancia (1 - similitud)
SELECT show_trgm('opencode');  -- muestra los trigramas generados
```

Junto con un indice GIN o GiST, permite busquedas eficientes mediante `ILIKE` y el operador `%`:

```sql
CREATE INDEX trgm_idx ON tabla USING GIN (columna gin_trgm_ops);
SELECT * FROM tabla WHERE columna % 'termino_busqueda';
```

### uuid-ossp

Genera identificadores UUID (Universally Unique Identifier) en sus variantes 1, 3, 4 y 5. Es fundamental para sistemas distribuidos donde las claves primarias autoincrementales causarian conflictos entre nodos.

```sql
CREATE EXTENSION "uuid-ossp";

SELECT uuid_generate_v1();      -- UUID basado en tiempo + MAC
SELECT uuid_generate_v4();      -- UUID aleatorio
SELECT uuid_nil();              -- UUID todo ceros
```

### btree_gist y btree_gin

Estas extensiones permiten crear indices GiST y GIN sobre tipos de datos que nativamente solo soportan indices B-tree. Su utilidad principal es habilitar restricciones de exclusion y consultas de rango en columnas escalares.

```sql
CREATE EXTENSION btree_gist;

-- Restriccion de exclusion: evitar solapamiento de rangos
CREATE TABLE reservas (
    sala_id INTEGER,
    periodo TSRANGE,
    EXCLUDE USING GIST (sala_id WITH =, periodo WITH &&)
);
```

### pgcrypto

Proporciona funciones criptograficas para hash, cifrado simetrico y asimetrico, y generacion de numeros aleatorios seguros. Es la extension de referencia para almacenar contrasenas y datos sensibles.

```sql
CREATE EXTENSION pgcrypto;

-- Hash de contrasena con algoritmo bcrypt
SELECT crypt('mi_contrasena', gen_salt('bf', 8));

-- Verificacion
SELECT crypt('mi_contrasena', hash_almacenado) = hash_almacenado;

-- Cifrado simetrico
SELECT encrypt('datos_secretos', 'clave', 'aes');
SELECT decrypt(dato_cifrado, 'clave', 'aes');
```

> `pgcrypto` es la opcion recomendada frente a funciones como `md5()` o `sha1()` integradas en el nucleo, ya que estas ultimas son rapidas pero inadecuadas para contrasenas por su resistencia a ataques de fuerza bruta. `crypt()` con `gen_salt('bf')` aplica bcrypt con factor de costo configurable.

### pg_stat_statements

Extension esencial para el ajuste de rendimiento. Registra estadisticas de ejecucion de todas las consultas SQL: numero de ejecuciones, tiempo total, tiempo de CPU, bloques leidos, filas afectadas, etc.

```sql
-- Compartir la extension entre bases de datos requiere carga en postgresql.conf
-- shared_preload_libraries = 'pg_stat_statements'

CREATE EXTENSION pg_stat_statements;

-- Consultas mas lentas
SELECT query, calls, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Requiere que la libreria compartida se cargue al arranque del servidor (`shared_preload_libraries`). Una vez activa, las estadisticas se acumulan hasta que se reinician manualmente:

```sql
SELECT pg_stat_statements_reset();
```

### postgres_fdw

__Foreign Data Wrapper__ para conectar con otras bases de datos PostgreSQL remotas. Se trata del FDW mas utilizado y se desarrolla en detalle en el archivo dedicado a wrappers.

```sql
CREATE EXTENSION postgres_fdw;
```

### file_fdw

Permite consultar archivos planos (CSV, text, logs) como si fueran tablas de PostgreSQL. Es util para cargar datos externos sin necesidad de ETL complejos.

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER file_server FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE logs (
    timestamp TIMESTAMP,
    nivel TEXT,
    mensaje TEXT
) SERVER file_server
OPTIONS (filename '/var/log/aplicacion.log', format 'csv');
```

### hstore

Implementa un tipo de datos clave-valor dentro de una columna. Precursor del soporte JSON nativo, sigue siendo util para esquemas flexibles con pares atributo-valor.

```sql
CREATE EXTENSION hstore;

CREATE TABLE productos (
    id SERIAL PRIMARY KEY,
    atributos HSTORE
);

INSERT INTO productos VALUES (1, 'color => rojo, peso => 1.5');

-- Acceso por clave
SELECT atributos -> 'color' FROM productos;
```

### ltree

Implementa el tipo de datos `ltree` para representar jerarquias etiquetadas, ideal para arboles de categorias, organigramas o taxonomias. Soporta operadores de descendencia, ascendencia y coincidencia de patrones.

```sql
CREATE EXTENSION ltree;

CREATE TABLE categorias (
    id SERIAL PRIMARY KEY,
    ruta LTREE
);

INSERT INTO categorias VALUES (1, 'root.educacion.programacion');
INSERT INTO categorias VALUES (2, 'root.educacion.redes');

-- Subarbol
SELECT * FROM categorias WHERE ruta ~ 'root.educacion.*';
```

### pg_partman

Automatiza la gestion de particionamiento por tiempo o por entero. Crea y mantiene particiones hijas automaticamente segun una politica configurada.

```sql
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
    p_parent_table := 'public.ventas',
    p_control := 'fecha',
    p_type := 'native',
    p_interval := '1 day',
    p_premake := 30
);
```

### pg_repack

Reorganiza tablas para eliminar la fragmentacion (_bloat_) sin necesidad de bloqueos exclusivos largos. Reemplaza operaciones como `VACUUM FULL` y `CLUSTER` en entornos donde no se puede interrumpir el acceso de escritura.

```sql
-- Instalacion: requiere binario en el sistema operativo
CREATE EXTENSION pg_repack;

-- Reorganizar una tabla sin bloqueo
SELECT pg_repack.repack_table('tabla_bloat');
```

### pgBadger

Herramienta externa de analisis de logs que genera informes HTML detallados sobre el rendimiento del servidor. No es una extension en si misma, sino un analizador de logs que se integra con PostgreSQL.

```bash
pgbadger /var/log/postgresql/postgresql.log -o report.html
```

### PostGIS

La extension geografica mas potente del ecosistema open source. Anade tipos `GEOMETRY` y `GEOGRAPHY`, funciones espaciales (ST_Distance, ST_Intersects, ST_Within), indices R-tree y soporte completo para el estandar OGC (Open Geospatial Consortium). Su uso es transversal a multiples sectores: logistica, urbanismo, agricultura de precision, analisis de redes.

```sql
CREATE EXTENSION postgis;

CREATE TABLE ubicaciones (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100),
    coordenadas GEOMETRY(Point, 4326)  -- SRID WGS84
);

INSERT INTO ubicaciones VALUES (1, 'Madrid', ST_SetSRID(ST_MakePoint(-3.703, 40.416), 4326));

SELECT ST_Distance(
    (SELECT coordenadas FROM ubicaciones WHERE id = 1),
    ST_SetSRID(ST_MakePoint(2.173, 41.385), 4326)  -- Barcelona
);
```

