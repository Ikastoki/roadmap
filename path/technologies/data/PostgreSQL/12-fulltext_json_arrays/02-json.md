# JSON y JSONB

PostgreSQL fue uno de los primeros motores relacionales en incorporar soporte nativo para JSON, y lo hizo con dos tipos distintos que responden a necesidades diferentes: `JSON` y `JSONB`. Comprender la diferencia entre ambos, asi como los operadores de extraccion, modificacion e indizacion, es esencial para trabajar con datos semiestructurados sin abandonar las ventajas del modelo relacional.

## JSON vs JSONB

El tipo `JSON` almacena el dato exactamente como se recibe, preservando el formato original, los espacios en blanco, el orden de las claves y las duplicaciones. Cada vez que se accede al valor, PostgreSQL debe parsearlo de nuevo, lo que introduce latencia en cada consulta.

`JSONB` (JSON Binary) almacena el dato en un formato binario interno optimizado. Al insertarse, el JSON se parsea una unica vez y se almacena sin espacios superfluos, con claves ordenadas y sin duplicados. El formato binario permite accesos directos sin re-parseo y, lo mas importante, soporta indizacion mediante GIN.

```sql
CREATE TABLE documentos (
    id SERIAL PRIMARY KEY,
    datos_json JSON,
    datos_jsonb JSONB
);

INSERT INTO documentos (datos_json, datos_jsonb) VALUES
    ('{"nombre": "Ana", "edad": 30}', '{"nombre": "Ana", "edad": 30}');
```

| Aspecto | JSON | JSONB |
|---------|------|-------|
| Almacenamiento | Texto plano (preserva formato) | Binario (normalizado) |
| Parseo | En cada operacion | Una vez al insertar |
| Claves duplicadas | Permite | Ultima prevalece |
| Orden de claves | Preserva | Orden alfabetico |
| Indices | No (solo expression index) | GIN, expression index |
| Velocidad de insercion | Rapida | Mas lenta (parseo) |
| Velocidad de consulta | Lenta | Rapida |

> Salvo que se necesite preservar el JSON original exactamente como llego (por ejemplo, por auditoria o validacion), siempre debe preferirse `JSONB`.

## Insercion de datos

Ambos tipos aceptan cadenas con formato JSON valido. La insercion puede hacerse con literales o mediante funciones de construccion:

```sql
INSERT INTO documentos (datos_jsonb) VALUES
    ('{"usuario": "jperez", "roles": ["admin", "editor"], "activo": true}');

-- Desde una variable o parametro
INSERT INTO documentos (datos_jsonb) VALUES
    ($1::jsonb);
```

## Extraccion de datos

PostgreSQL ofrece una familia de operadores para acceder a los elementos internos de un JSON.

### Operador ->

El operador `->` extrae un campo como tipo `JSON` (o `JSONB` segun el tipo base). Devuelve el valor intacto, incluyendo sus comillas si es cadena.

```sql
SELECT datos_jsonb -> 'usuario' FROM documentos;
-- "jperez"

SELECT datos_jsonb -> 'roles' FROM documentos;
-- ["admin", "editor"]

SELECT datos_jsonb -> 'roles' -> 0 FROM documentos;
-- "admin"
```

### Operador ->>

`->>` extrae el valor como texto (`TEXT`), eliminando las comillas en cadenas. Es el operador mas usado en proyecciones de consultas.

```sql
SELECT datos_jsonb ->> 'usuario' FROM documentos;
-- jperez

SELECT datos_jsonb ->> 'roles' FROM documentos;
-- ["admin", "editor"]  (sigue siendo texto, no array)
```

### Operadores #> y #>>

Estos operadores permiten navegar por rutas anidadas sin encadenar multiples `->`. La ruta se expresa como un array de texto.

```sql
SELECT datos_jsonb #> '{roles, 0}' FROM documentos;
-- "admin"

SELECT datos_jsonb #>> '{roles, 0}' FROM documentos;
-- admin
```

## Operadores de existencia y contencion

JSONB (no JSON) soporta operadores para consultar la estructura del documento sin conocer su esquema exacto.

### ?

El operador `?` comprueba si existe una clave en el nivel raiz.

```sql
SELECT datos_jsonb ? 'usuario' FROM documentos;
-- true
```

### ?| y ?&

`?|` comprueba si alguna de las claves existe; `?&` comprueba si todas existen.

```sql
SELECT datos_jsonb ?| ARRAY['usuario', 'email'] FROM documentos;
-- true (usuario existe)

SELECT datos_jsonb ?& ARRAY['usuario', 'activo'] FROM documentos;
-- true (ambas existen)
```

### @> y <@

`@>` comprueba si el JSONB de la izquierda contiene al de la derecha (subconjunto). `<@` es la operacion inversa (el de la izquierda esta contenido en el de la derecha).

```sql
SELECT datos_jsonb @> '{"usuario": "jperez"}'::jsonb FROM documentos;
-- true

SELECT datos_jsonb @> '{"roles": ["admin"]}'::jsonb FROM documentos;
-- true (coincidencia parcial del array)

SELECT '{"usuario": "jperez"}'::jsonb <@ datos_jsonb FROM documentos;
-- true
```

## Modificacion de JSONB

### Operador ||

El operador `||` fusiona dos JSONB. Si hay claves duplicadas, el valor de la derecha prevalece.

```sql
SELECT datos_jsonb || '{"email": "jperez@ejemplo.com"}'::jsonb FROM documentos;
-- {"usuario": "jperez", "roles": ["admin", "editor"], "activo": true, "email": "jperez@ejemplo.com"}
```

### jsonb_set

`jsonb_set(target, path, new_value, create_if_missing)` actualiza un valor en una ruta especifica.

```sql
UPDATE documentos
SET datos_jsonb = jsonb_set(datos_jsonb, '{activo}', 'false'::jsonb, false);

UPDATE documentos
SET datos_jsonb = jsonb_set(datos_jsonb, '{direccion, ciudad}', '"Madrid"'::jsonb, true);
```

### jsonb_insert

`jsonb_insert(target, path, new_value, after_flag)` inserta un nuevo valor sin sobrescribir. Si `after_flag` es `true`, inserta despues de la posicion en arrays; si es `false`, antes.

```sql
UPDATE documentos
SET datos_jsonb = jsonb_insert(datos_jsonb, '{roles, 1}', '"lector"'::jsonb, false);
-- Inserta "lector" en la posicion 1 del array roles
```

### jsonb_object_keys

Devuelve el conjunto de claves del nivel raiz como filas:

```sql
SELECT jsonb_object_keys(datos_jsonb) AS clave FROM documentos;
```

### jsonb_array_length y jsonb_array_elements

```sql
SELECT jsonb_array_length(datos_jsonb -> 'roles') FROM documentos;
-- 3

SELECT jsonb_array_elements(datos_jsonb -> 'roles') AS rol FROM documentos;
-- Devuelve una fila por elemento del array
```

## Indices GIN para JSONB

La capacidad de indizar JSONB diferencia a PostgreSQL de la mayoria de bases de datos relacionales. Un indice GIN sobre una columna JSONB acelera los operadores `?`, `?|`, `?&`, `@>` y `<@`.

```sql
CREATE INDEX idx_documentos_jsonb ON documentos USING GIN (datos_jsonb);
```

### jsonb_path_ops

Por defecto, GIN usa la clase de operadores `jsonb_ops`, que extrae cada clave y valor como un token independiente. La clase `jsonb_path_ops` extrae solo las rutas completas (hash de clave+valor), generando un indice mas pequeno y rapido para `@>`.

```sql
CREATE INDEX idx_documentos_path ON documentos USING GIN (datos_jsonb jsonb_path_ops);
```

La desventaja de `jsonb_path_ops` es que solo acelera `@>` y `<@`, no los operadores `?`, `?|` ni `?&`.

### Expression index

Cuando las consultas acceden siempre a una clave especifica del JSONB, un indice sobre esa ruta concreta suele ser mas eficiente que un GIN general:

```sql
CREATE INDEX idx_documentos_usuario ON documentos USING BTREE ((datos_jsonb ->> 'usuario'));
```

Este indice acelera consultas como `WHERE datos_jsonb ->> 'usuario' = 'jperez'`, mientras que un GIN no beneficiaria esa busqueda exacta sobre un campo concreto.

