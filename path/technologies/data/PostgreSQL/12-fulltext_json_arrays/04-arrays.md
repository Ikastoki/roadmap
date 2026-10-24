# Arrays

PostgreSQL es el unico motor relacional maduro que ofrece arrays como tipo de dato nativo de primera clase, con sintaxis dedicada, operadores especificos y soporte de indices. Lejos de ser una simple curiosidad sintactica, los arrays resuelven problemas reales de modelado: relaciones uno-a-muchos sin tabla intermedia, almacenamiento de secuencias ordenadas y operaciones de pertenencia en memoria.

## Tipo array

Cualquier tipo de dato en PostgreSQL puede convertirse en array anadiendo `[]` al nombre del tipo. Los arrays pueden ser de tipos nativos (`INTEGER[]`, `TEXT[]`), de tipos definidos por el usuario o incluso de tipos compuestos.

```sql
CREATE TABLE productos (
    id SERIAL PRIMARY KEY,
    nombre TEXT NOT NULL,
    etiquetas TEXT[],
    precios_historicos NUMERIC[]
);
```

### Notacion con ARRAY

Los literales array pueden expresarse con la funcion constructora `ARRAY`:

```sql
INSERT INTO productos (nombre, etiquetas, precios_historicos) VALUES
    ('Laptop', ARRAY['electronica', 'ofertas'], ARRAY[999.99, 899.99, 749.99]);
```

Alternativamente, usando la notacion de cadena con llaves:

```sql
INSERT INTO productos (nombre, etiquetas) VALUES
    ('Teclado', '{accesorios, electronica}');
```

> La notacion con `ARRAY[...]` es mas legible y menos propensa a errores de escape que la notacion con llaves.

### Arrays multidimensionales

PostgreSQL soporta arrays de cualquier dimensionalidad. Se declaran anidando corchetes y se construyen con `ARRAY` anidado:

```sql
CREATE TABLE matrices (
    id SERIAL PRIMARY KEY,
    datos INTEGER[][]
);

INSERT INTO matrices (datos) VALUES
    (ARRAY[[1, 2, 3], [4, 5, 6], [7, 8, 9]]);
```

Para acceder a elementos multidimensionales se usa la sintaxis `[dim1][dim2]`:

```sql
SELECT datos[2][3] FROM matrices;  -- 6
```

## Operaciones con arrays

### Acceso por posicion

Los arrays en PostgreSQL usan indexacion basada en 1 (no en 0). El acceso fuera de los limites devuelve NULL, no lanza error.

```sql
SELECT etiquetas[1] FROM productos;       -- 'electronica'
SELECT etiquetas[3] FROM productos;       -- NULL
```

### array_append y array_prepend

Anaden un elemento al final o al principio del array, respectivamente.

```sql
UPDATE productos
SET etiquetas = array_append(etiquetas, 'nuevo');

UPDATE productos
SET etiquetas = array_prepend('destacado', etiquetas);
```

### array_cat

Concatena dos arrays en uno nuevo.

```sql
UPDATE productos
SET etiquetas = array_cat(etiquetas, ARRAY['verano', 'liquidacion']);
```

### array_remove y array_replace

`array_remove` elimina todas las ocurrencias de un valor. `array_replace` sustituye todas las ocurrencias de un valor por otro.

```sql
SELECT array_remove(ARRAY[1, 2, 3, 2, 1], 2);
-- {1, 3, 1}

SELECT array_replace(ARRAY[1, 2, 3, 2], 2, 99);
-- {1, 99, 3, 99}
```

## Funciones de metadatos

### array_length y array_upper

`array_length(array, dimension)` devuelve la longitud de una dimension especifica. `array_upper` devuelve el indice maximo.

```sql
SELECT array_length(etiquetas, 1) FROM productos;
-- Numero de etiquetas

SELECT array_length(ARRAY[[1,2,3],[4,5,6]], 2);
-- 3 (longitud de la segunda dimension)
```

### cardinality

`cardinality` devuelve el numero total de elementos en el array, independientemente de su dimensionalidad.

```sql
SELECT cardinality(ARRAY[[1,2,3],[4,5,6]]);
-- 6
```

### array_dims

Devuelve una cadena con los limites de cada dimension:

```sql
SELECT array_dims(ARRAY[[1,2,3],[4,5,6]]);
-- [1:2][1:3]
```

## unnest

`unnest` es la funcion mas importante para trabajar con arrays en SQL. Expande un array en un conjunto de filas, una por elemento. Permite que los arrays interactuen con el modelo relacional.

```sql
SELECT id, unnest(etiquetas) AS etiqueta
FROM productos;

-- 1 | electronica
-- 1 | ofertas
-- 2 | accesorios
-- 2 | electronica
```

Con arrays multiples, `unnest` expande en paralelo:

```sql
SELECT unnest(ARRAY[1, 2, 3]) AS id, unnest(ARRAY['a', 'b', 'c']) AS letra;
-- 1 | a
-- 2 | b
-- 3 | c
```

Cuando los arrays tienen distinta longitud, las posiciones faltantes se rellenan con NULL.

## ANY y ALL

Los constructores `ANY` y `ALL` permiten comparar un valor escalar con todos los elementos de un array.

### ANY

`ANY` devuelve `true` si al menos un elemento del array cumple la condicion:

```sql
SELECT 'electronica' = ANY(etiquetas) FROM productos;
-- true para productos etiquetados como electronica

SELECT * FROM productos WHERE 'ofertas' = ANY(etiquetas);
```

### ALL

`ALL` devuelve `true` si todos los elementos cumplen la condicion:

```sql
SELECT 50 > ALL(ARRAY[10, 20, 30]);
-- true

SELECT 50 > ALL(ARRAY[10, 60, 30]);
-- false
```

## Operadores de contencion para arrays

Al igual que JSONB, los arrays tienen operadores de contencion, aunque solo funcionan con arrays unidimensionales.

### @> y <@

`@>` comprueba si el array izquierdo contiene al derecho. `<@` es la operacion inversa.

```sql
SELECT ARRAY['a', 'b', 'c'] @> ARRAY['a', 'c'];
-- true

SELECT ARRAY['a', 'c'] <@ ARRAY['a', 'b', 'c'];
-- true
```

### &&

`&&` comprueba si dos arrays tienen al menos un elemento en comun (interseccion no vacia).

```sql
SELECT ARRAY['a', 'b', 'c'] && ARRAY['c', 'd', 'e'];
-- true

SELECT * FROM productos WHERE etiquetas && ARRAY['ofertas', 'nuevo'];
```

## Indices GIN para arrays

Un indice GIN sobre una columna array acelera los operadores `@>`, `<@`, `&&` y `= ANY()`.

```sql
CREATE INDEX idx_productos_etiquetas ON productos USING GIN (etiquetas);
```

Las consultas que usan `@>` o `&&` pasan de escaneos secuenciales a busquedas directas en el indice invertido:

```sql
EXPLAIN ANALYZE SELECT * FROM productos WHERE etiquetas @> ARRAY['electronica'];
```

> Los indices GIN sobre arrays son especialmente efectivos cuando los arrays tienen muchos elementos distintos y las consultas filtran por combinaciones de ellos (tags, categorias, permisos).

A diferencia de GIN sobre JSONB, los indices sobre arrays no requieren configuracion adicional; la clase de operadores por defecto ya soporta todas las operaciones relevantes.

### Consideraciones de rendimiento

- Los arrays con muchos elementos (>100) incrementan el tamano del indice GIN y ralentizan las inserciones.
- Para arrays pequenos (<10 elementos) que se consultan con frecuencia, considerar normalizar en una tabla separada.
- El parametro `gin_pending_list_limit` controla la memoria para inserciones diferidas en GIN con `fastupdate` activado.

