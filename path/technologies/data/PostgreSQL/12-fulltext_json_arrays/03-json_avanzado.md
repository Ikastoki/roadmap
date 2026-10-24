# Funciones avanzadas para JSON

El soporte JSON de PostgreSQL no se limita a almacenar y consultar documentos. Incluye un conjunto de funciones que permiten convertir filas relacionales en documentos JSON y viceversa, posibilitando la integracion directa con APIs REST, sistemas basados en documentos y formatos de intercambio sin necesidad de capas intermedias de transformacion.

## json_populate_record

`json_populate_record` toma un tipo de fila (definido mediante `CREATE TYPE` o una referencia de tabla `%TYPE`) y un JSON, y devuelve una fila tipada donde cada clave del JSON se asigna a una columna del registro. Las claves faltantes se rellenan con `NULL`; las claves sobrantes se ignoran.

```sql
CREATE TYPE usuario_tipo AS (
    id INTEGER,
    nombre TEXT,
    email TEXT
);

SELECT * FROM json_populate_record(
    NULL::usuario_tipo,
    '{"id": 1, "nombre": "Ana Lopez", "email": "ana@ejemplo.com"}'
);
```

Cuando se usa con una tabla existente en lugar de un tipo explicito:

```sql
SELECT * FROM json_populate_record(
    NULL::usuarios,
    '{"id": 1, "nombre": "Ana Lopez", "email": "ana@ejemplo.com"}'
);
```

La variante `jsonb_populate_record` funciona identicamente con `JSONB`.

### json_populate_recordset

La variante `json_populate_recordset` procesa un array de objetos JSON y devuelve un conjunto de filas, equivalente a un `UNNEST` combinado con `json_populate_record`:

```sql
SELECT * FROM json_populate_recordset(
    NULL::usuario_tipo,
    '[{"id": 1, "nombre": "Ana"}, {"id": 2, "nombre": "Luis"}]'
);
```

Esta funcion es particularmente util para recibir `POST` o `PUT` con listas de entidades y realizar operaciones masivas.

## row_to_json

`row_to_json` convierte una fila de una tabla o consulta en un objeto JSON. Cada columna se convierte en una clave con su nombre y valor:

```sql
SELECT row_to_json(u) FROM usuarios u WHERE id = 1;
-- {"id": 1, "nombre": "Ana Lopez", "email": "ana@ejemplo.com"}
```

La funcion acepta un segundo parametro booleano `pretty_bool` que, si es `true`, genera una salida con formato legible (saltos de linea e indentacion).

```sql
SELECT row_to_json(u, true) FROM usuarios u WHERE id = 1;
```

Para controlar las claves incluidas en el JSON, se usa una subconsulta o una fila anonima:

```sql
SELECT row_to_json(t) FROM (
    SELECT id, nombre FROM usuarios WHERE activo = true
) t;
```

### to_json y to_jsonb

`to_json` (y su contraparte `to_jsonb`) convierten cualquier valor SQL a su representacion JSON. A diferencia de `row_to_json`, que opera sobre filas completas, `to_json` se usa para valores individuales:

```sql
SELECT to_jsonb(current_timestamp);
-- "2026-07-28T12:00:00+02:00"

SELECT to_jsonb(ARRAY[1, 2, 3]);
-- [1, 2, 3]
```

## json_agg

`json_agg` es una funcion de agregacion que construye un array JSON a partir de las filas del grupo. Es el equivalente de `ARRAY_AGG` pero para JSON:

```sql
SELECT json_agg(u) FROM usuarios u;
-- [{"id": 1, "nombre": "Ana Lopez", ...}, {"id": 2, "nombre": "Luis Perez", ...}]
```

Combinado con `GROUP BY`, permite construir documentos JSON anidados directamente desde SQL relacional:

```sql
SELECT d.nombre AS departamento,
       json_agg(json_build_object('id', e.id, 'nombre', e.nombre)) AS empleados
FROM departamentos d
JOIN empleados e ON e.depto_id = d.id
GROUP BY d.id, d.nombre;
```

El resultado es un JSON perfectamente estructurado para servir directamente como respuesta de API.

### json_build_object y json_build_array

`json_build_object` construye un objeto JSON a partir de pares clave-valor:

```sql
SELECT json_build_object('id', 1, 'nombre', 'Ana', 'roles', json_build_array('admin', 'editor'));
-- {"id": 1, "nombre": "Ana", "roles": ["admin", "editor"]}
```

### json_object_agg

`json_object_agg` construye un objeto JSON desde una consulta que devuelve pares clave-valor:

```sql
SELECT json_object_agg(usuario, email) FROM usuarios;
-- {"Ana Lopez": "ana@ejemplo.com", "Luis Perez": "luis@ejemplo.com"}
```

## json_each y json_object_keys

`json_each` expande un objeto JSON en un conjunto de filas con dos columnas: `key` (TEXT) y `value` (JSON). Su variante `json_each_text` devuelve `value` como TEXT.

```sql
SELECT * FROM json_each('{"a": 1, "b": "texto", "c": [1,2,3]}'::json);
-- key |  value
-- a   |  1
-- b   |  "texto"
-- c   |  [1,2,3]
```

`json_object_keys` es mas simple: devuelve solo las claves del objeto.

```sql
SELECT json_object_keys('{"a": 1, "b": 2}'::json);
-- a
-- b
```

## json_array_elements

`json_array_elements` expande un array JSON en filas. Cada elemento se devuelve como tipo JSON.

```sql
SELECT json_array_elements('[1, "dos", {"tres": 3}]'::json);
-- 1
-- "dos"
-- {"tres": 3}
```

La variante `json_array_elements_text` devuelve los elementos como TEXT.

## json_typeof

Devuelve el tipo del valor JSON como cadena: `string`, `number`, `boolean`, `object`, `array` o `null`.

```sql
SELECT json_typeof('{"a": 1}'::json -> 'a');
-- number
```

## Integracion con APIs

La combinacion de `row_to_json` y `json_agg` permite construir respuestas completas para APIs REST sin procesamiento externo:

```sql
SELECT json_build_object(
    'status', 'ok',
    'data', json_agg(
        json_build_object(
            'id', p.id,
            'nombre', p.nombre,
            'precio', p.precio,
            'categoria', (SELECT row_to_json(c) FROM categorias c WHERE c.id = p.categoria_id)
        )
    ),
    'total', (SELECT count(*) FROM productos)
) AS respuesta
FROM productos p;
```

Esta consulta devuelve un unico JSON perfectamente formateado que puede enviarse directamente como respuesta HTTP desde una aplicacion que use PostgreSQL como capa de datos.

