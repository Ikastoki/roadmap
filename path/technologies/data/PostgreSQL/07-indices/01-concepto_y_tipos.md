# Indices

Un indice es una estructura auxiliar que acelera la localizacion de filas en una tabla, operando de forma similar al indice tematico de un libro: en lugar de hojear pagina por pagina (un `full table scan`), se consulta una estructura ordenada que indica donde encontrar cada valor. Sin indice, una busqueda por una columna recorre toda la tabla con coste O(n); con un indice B-Tree, el coste se reduce a O(log n) en el numero de filas. Esta diferencia es insignificante en tablas con cientos de filas, pero se vuelve determinante en tablas con millones de registros.

El beneficio no es gratuito: cada indice ocupa espacio en disco y ralentiza las operaciones de escritura (`INSERT`, `UPDATE`, `DELETE`), puesto que cada modificacion debe reflejarse tanto en la tabla como en el indice. Por tanto, conviene crear indices solo cuando el beneficio en lectura justifica el coste en escritura y almacenamiento. Las columnas que aparecen en clausulas `WHERE`, `JOIN` y `ORDER BY` son las candidatas naturales; las columnas con alta cardinalidad (muchos valores distintos) se benefician mas que las que tienen pocos valores (como un booleano). En tabolas pequenas o con workloads predominantemente de escritura, los indices pueden ser contraproducentes.

PostgreSQL ofrece una variedad de tipos de indice, cada uno optimizado para patrones de acceso distintos. Elegir el tipo adecuado puede marcar una diferencia de ordenes de magnitud en el rendimiento de las consultas.

## B-Tree

El B-Tree es el tipo por defecto y el mas versatil. Es un arbol balanceado en el que las hojas contienen los valores de la columna indexada junto con punteros a las filas correspondientes. Soporta operaciones de igualdad (`=`), comparacion (`<`, `<=`, `>`, `>=`), rango (`BETWEEN`), busqueda de prefijos (`LIKE 'patron%'`) y ordenacion (`ORDER BY`).

```sql
CREATE INDEX idx_usuarios_email ON usuarios(email);
```

PostgreSQL puede recorrer el arbol en ambos sentidos, por lo que un solo indice B-Tree cubre tanto `ORDER BY ASC` como `ORDER BY DESC` con el mismo coste logaritmico. Cuando el indice contiene todas las columnas necesarias para una consulta, se produce un `Index Only Scan` que evita acceder a la tabla por completo.

El B-Tree maneja NULLs de forma configurable: por defecto los considera mayores que cualquier valor no nulo, pero esto puede alterarse con las clausulas `NULLS FIRST` o `NULLS LAST` en la creacion del indice.

## Hash

El indice Hash esta disenado exclusivamente para operaciones de igualdad simple (`=`). Internamente almacena un codigo hash calculado a partir del valor de la columna, y las busquedas se resuelven en tiempo constante O(1) en promedio, superando al B-Tree en este escenario concreto.

```sql
CREATE INDEX idx_sesiones_token ON sesiones USING hash(token);
```

Su utilidad es limitada porque no soporta comparaciones de rango, ordenacion ni busqueda por patron. En la mayoria de las aplicaciones, un B-Tree ofrece rendimiento comparable para igualdad sin perder la versatilidad, por lo que los indices Hash se reservan para casos donde la igualdad domina absolutamente el patron de acceso y la tabla es demasiado grande para que el B-Tree sea eficiente.

> Hasta PostgreSQL 10, los indices Hash no estaban protegidos por WAL y podian corromperse tras un crash. Desde la version 10 son completamente seguros y replicables.

## GiST

GiST (Generalized Search Tree) es un arbol de busqueda balanceado y extensible que permite implementar operadores de busqueda mas alla de la igualdad y el orden: interseccion, contencion, proximidad espacial y busqueda por similitud. Funciona como un framework sobre el que se construyen dominios especificos.

```sql
CREATE INDEX idx_ubicaciones ON lugares USING gist(ubicacion);
```

Los casos de uso mas habituales son datos geometricos (puntos, poligonos, circulos) con operadores como `&&` (solapamiento), `@>` (contenencia) y `~=` (coincidencia). GiST tambien se utiliza para `full-text search` como alternativa a GIN, aunque en ese escenario GIN suele ofrecer mejor rendimiento en lecturas.

```sql
-- Busqueda geometrica: puntos dentro de un poligono
SELECT * FROM lugares WHERE ubicacion && 'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::polygon;
```

Una ventaja practica de GiST es que, al ser extensible, permite que extensiones como PostGIS anadan sus propios operadores y metodos de indexacion sin modificar el nucleo de PostgreSQL.

## SP-GiST

SP-GiST (Space-Partitioned GiST) es una variante de GiST optimizada para datos que no se distribuyen uniformemente en el espacio de busqueda. En lugar de dividir el espacio de forma equitativa como hace un B-Tree, SP-GiST particiona recursivamente el espacio segun la distribucion real de los datos, lo que resulta especialmente eficiente para estructuras jerarquicas, arborescencias y datos con clustering natural.

```sql
CREATE INDEX idx_arbol_gen ON personas USING spgist(apellido);
```

SP-GiST es particularmente util para indices sobre arboles (extension `ltree`), busquedas geograficas con distribucion irregular y ciertos tipos de datos de texto cuando los patrones de busqueda no siguen un prefijo fijo. En terminos generales, si GiST se comporta bien pero los datos tienen una densidad muy variable, SP-GiST puede ofrecer un rendimiento superior.

## GIN

GIN (Generalized Inverted Index) esta disenado para valores compuestos: columnas que contienen multiples elementos dentro de una unica fila. En lugar de mapear cada fila a un valor, GIN construye un indice inverso que mapea cada elemento individual a la lista de filas que lo contienen.

```sql
CREATE INDEX idx_productos_etiquetas ON productos USING gin(etiquetas);
```

GIN es el tipo indicado para arrays (`TEXT[]`, `INTEGER[]`), documentos JSONB y busqueda de texto completo (`tsvector`). Soporta los operadores `@>` (contiene), `<@` (contenido en), `?` (tiene clave), `?|` (alguna clave) y `?&` (todas las claves).

```sql
-- Busqueda en JSONB
SELECT * FROM productos WHERE datos @> '{"color": "rojo"}';

-- Full-text search con GIN
SELECT * FROM documentos WHERE to_tsvector('spanish', contenido) @@ to_tsquery('spanish', 'gato & perro');
```

GIN tiene un costo de escritura mayor que B-Tree porque mantiene listas de entradas para cada elemento. Para mitigarlo, PostgreSQL permite configurar `fastupdate` (insercion en buffer antes de la actualizacion masiva del indice) y ajustar `gin_pending_list_limit`.

> GIN con `jsonb_path_ops` reduce el tamano del indice y acelera las busquedas con el operador `@>` a costa de no soportar `?`, `?|` ni `?&`.

## BRIN

BRIN (Block Range INdex) funciona especialmente bien en tablas muy grandes donde los datos estan ordenados fisicamente por una columna (por ejemplo, un `created_at` en una tabla de logs). En lugar de indexar cada fila, BRIN divide la tabla en bloques de paginas consecutivas y almacena el valor minimo y maximo de cada rango.

```sql
CREATE INDEX idx_logs_fecha ON logs USING brin(created_at) WITH (pages_per_range = 32);
```

El resultado es un indice extremadamente compacto: un BRIN puede ocupar cientos de kilobytes donde un B-Tree ocuparia gigabytes. A cambio, las busquedas no son exactas: PostgreSQL escanea los rangos que podrian contener el valor buscado y luego examina las paginas de esos rangos, lo que se conoce como `Bitmap Index Scan`.

`pages_per_range` controla cuantas paginas agrupa cada rango. Valores mas altos reducen el tamano del indice pero aumentan la probabilidad de falsos positivos (rangos que se escanean sin necesidad). El valor por defecto es 128.

BRIN es la eleccion natural para tablas de tipo serie temporal, logs de auditoria, datos de sensores o cualquier escenario donde las inserciones sean secuenciales y las consultas abarquen rangos de tiempo acotados.

