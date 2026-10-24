# Full-Text Search

Un motor de base de datos relacional clasico busca coincidencias exactas con `LIKE` o expresiones regulares, operaciones que recorren secuencialmente los caracteres. PostgreSQL va mas alla con su sistema de busqueda textual (Full-Text Search, FTS), un subsistema especializado que analiza el lenguaje natural, reduce las palabras a su raiz lexica y construye indices semanticos capaces de responder que documentos contienen que conceptos, no solo que cadenas.

## tsvector y tsquery

Los dos tipos de datos fundamentales del FTS son `tsvector` y `tsquery`. Un `tsvector` es una representacion normalizada de un texto: una lista ordenada de lexemas (unidades lexicas) con las posiciones en las que aparecen. Cada lexema ha pasado por un proceso de stemming que elimina variaciones morfologicas (plurales, conjugaciones, generos).

```sql
SELECT to_tsvector('spanish', 'Los gatos caminan silenciosamente por el tejado');
-- 'gato':2 'camin':3 'silencios':4 'tejad':6
```

Un `tsquery` expresa una condicion de busqueda: uno o varios lexemas combinados con operadores logicos `&` (AND), `|` (OR) y `!` (NOT), ademas del operador de frase `<->` con distancia.

```sql
SELECT to_tsquery('spanish', 'gato & caminar');
-- 'gat' & 'camin'
```

## Funciones de conversion

`to_tsvector(config, texto)` toma un texto y devuelve su representacion como `tsvector` aplicando la configuracion lingueistica especificada. `to_tsquery(config, texto)` normaliza una cadena de busqueda a `tsquery`. La configuracion determina el idioma, el diccionario y las stop words.

```sql
SELECT to_tsvector('english', 'running runners run');
-- 'run':1,2,3

SELECT to_tsquery('english', 'running & runner');
-- 'run' & 'run'
-- Se normaliza a 'run' & 'run', que equivale a 'run'
```

`plainto_tsquery` convierte texto plano en `tsquery` insertando automaticamente el operador `&` entre las palabras, sin necesidad de escribir los operadores logicos manualmente.

```sql
SELECT plainto_tsquery('spanish', 'gatos caminan tejado');
-- 'gat' & 'camin' & 'tejad'
```

## Operador @@

El operador `@@` evalua si un `tsvector` satisface un `tsquery`. Es el nucleo del FTS: combina el texto indizado con la condicion de busqueda.

```sql
SELECT to_tsvector('spanish', 'El gato camina') @@ to_tsquery('spanish', 'gato');
-- true
```

En consultas sobre tablas:

```sql
CREATE TABLE articulos (
    id SERIAL PRIMARY KEY,
    titulo TEXT,
    cuerpo TEXT
);

INSERT INTO articulos (titulo, cuerpo) VALUES
    ('PostgreSQL avanzado', 'Las funciones ventana permiten analizar series temporales'),
    ('Indices en PostgreSQL', 'Los indices GIN aceleran busquedas de texto completo');

SELECT id, titulo FROM articulos
WHERE to_tsvector('spanish', titulo || ' ' || cuerpo) @@ plainto_tsquery('spanish', 'indices busquedas');
```

## Ranking de resultados

`ts_rank` y `ts_rank_cd` calculan la relevancia de un documento respecto a una consulta. El ranking considera frecuencia de los terminos, proximidad entre ellos y densidad en el documento.

```sql
SELECT id, titulo,
       ts_rank(to_tsvector('spanish', titulo || ' ' || cuerpo),
               plainto_tsquery('spanish', 'indices GIN')) AS rank
FROM articulos
WHERE to_tsvector('spanish', titulo || ' ' || cuerpo) @@ plainto_tsquery('spanish', 'indices GIN')
ORDER BY rank DESC;
```

`ts_rank` devuelve un valor numerico entre 0 y 1. La variante `ts_rank_cd` utiliza densidad de portada (cover density), que pondera mas los terminos cercanos entre si.

## Configuracion lingueistica

PostgreSQL incluye configuraciones para decenas de idiomas. Cada configuracion define:

- **Parser**: divide el texto en tokens (palabras, URLs, direcciones email, numeros).
- **Diccionarios**: asignan cada token a uno o varios lexemas. Los tipos principales son:
  - `simple`: conversion a minusculas sin stemming.
  - `stem`: aplica stemming basado en el algoritmo Snowball.
  - `ispell`, `hunspell`: diccionarios externos para mas precision.
  - `synonym`: mapea sinonimos.
  - `thesaurus`: expansion de terminos jerarquicos.

```sql
-- Ver configuracion actual
\dF spanish

-- Ver diccionarios disponibles
SELECT * FROM pg_ts_dict;

-- Ver stop words del espanol
SELECT * FROM ts_stat('SELECT to_tsvector(''spanish'', cuerpo) FROM articulos')
ORDER BY nentry DESC LIMIT 10;
```

### Stop words

Las stop words son palabras vacias (articulos, preposiciones, conjunciones) que se eliminan durante la tokenizacion porque no aportan valor semantico a la busqueda. Cada configuracion lingueistica tiene su propia lista de stop words.

```sql
SELECT to_tsvector('spanish', 'El gato y el perro juegan en el patio');
-- 'gato':2 'perr':5 'jueg':6 'pati':8
-- 'el', 'y', 'en' desaparecen
```

### Stemming

El stemming reduce las palabras a su raiz o lexema. En espanol, "corriendo", "corria" y "correran" se reducen a "corr". Esto permite que una busqueda de "correr" encuentre documentos que contengan cualquier forma conjugada.

```sql
SELECT to_tsvector('spanish', 'corriendo corria correran corre');
-- 'corr':1,2,3,4
```

## Indices GIN vs GiST

Sin un indice, cada consulta FTS debe escanear todas las filas y calcular `to_tsvector` en cada una. Para tablas grandes es inviable. PostgreSQL ofrece dos tipos de indice especificos para FTS:

### GIN (Generalized Inverted Index)

GIN construye un indice invertido: por cada lexema almacena la lista de filas donde aparece. Es el indice recomendado para FTS porque:

- Las busquedas son muy rapidas (acceso directo por lexema).
- La creacion del indice es mas lenta que GiST.
- Ocupa mas espacio en disco.
- Soporta busquedas con operadores `&&`, `@>`, `<@`.

```sql
CREATE INDEX idx_articulos_fts ON articulos USING GIN (to_tsvector('spanish', titulo || ' ' || cuerpo));
```

GIN tiene tres configuraciones de velocidad: `gin_fuzzy_search_limit` limita el numero de resultados para evitar lecturas excesivas; el parametro `fastupdate` (activado por defecto) usa una lista de insercion diferida que mejora el rendimiento en escrituras intensivas.

### GiST (Generalized Search Tree)

GiST es un arbol de busqueda balanceado. Para FTS es menos eficiente que GIN en consultas, pero ofrece ventajas:

- Menor ocupacion de espacio.
- Insercion mas rapida.
- Adecuado en entornos con alta tasa de escritura y pocas consultas.

```sql
CREATE INDEX idx_articulos_fts_gist ON articulos USING GiST (to_tsvector('spanish', titulo || ' ' || cuerpo));
```

| Aspecto | GIN | GiST |
|---------|-----|------|
| Velocidad de consulta | Muy rapida | Lenta |
| Velocidad de insercion | Lenta (con fastupdate mejora) | Rapida |
| Espacio en disco | Mayor | Menor |
| Uso recomendado | Lecturas frecuentes | Escrituras frecuentes |

> Para la mayoria de aplicaciones con FTS, GIN es la opcion correcta. GiST solo se justifica cuando la tabla recibe multiples inserciones por segundo y las consultas de busqueda son esporadicas.

### Indices combinados

Es posible crear un indice GIN sobre una columna `tsvector` almacenada, evitando recalcular `to_tsvector` en cada insercion y consulta:

```sql
ALTER TABLE articulos ADD COLUMN texto_busqueda tsvector
    GENERATED ALWAYS AS (to_tsvector('spanish', titulo || ' ' || cuerpo)) STORED;

CREATE INDEX idx_articulos_fts_col ON articulos USING GIN (texto_busqueda);
```

Con esta estrategia, el indice se actualiza automaticamente al modificar las columnas base y las consultas no necesitan la funcion `to_tsvector`, simplificando el SQL y mejorando el rendimiento.

