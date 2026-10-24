# Herramientas de administracion

MongoDB proporciona un ecosistema de herramientas que complementan al shell para tareas de visualizacion, importacion, exportacion, monitorizacion y analisis de rendimiento. Cada herramienta esta disenada para un proposito especifico, y conocer su uso permite al administrador abordar desde inspecciones visuales hasta automatizaciones de backup sin depender exclusivamente de consultas manuales.

## MongoDB Compass

Compass es la interfaz grafica oficial de MongoDB. Ofrece una vista estructurada de los datos y herramientas de analisis que facilitan la comprension del esquema y el rendimiento de las consultas.

### Conexion y exploracion

Compass acepta cadenas de conexion estandar de MongoDB. Una vez conectado, muestra el arbol de bases de datos y colecciones en el panel izquierdo.

La interfaz permite:
- Navegar documentos con paginacion y filtros visuales
- Editar documentos in-place
- Ordenar y proyectar campos sin escribir consultas
- Visualizar indices y sus propiedades

### Schema analysis

La pestana **Schema** analiza una muestra de documentos y genera un resumen estadistico de cada campo:

- Tipos de datos detectados (string, int, double, array, object, etc.)
- Distribucion de valores (min, max, valores mas frecuentes)
- Porcentaje de documentos que contienen cada campo
- Longitud de strings y tamanos de arrays

Este analisis es util para detectar inconsistencia de esquema: si un campo aparece como string en el 70% de los documentos y como int en el 30%, hay un problema de modelado que debe corregirse.

### Performance

La pestana **Explain Plan** ejecuta una consulta y muestra su plan de ejecucion grafico:

```javascript
// Consulta en Compass
db.pedidos.find({ total: { $gt: 100 } }).sort({ fecha: -1 }).limit(50)
```

Compass visualiza:
- **COLLSCAN**: recorrido completo de la coleccion (sin indice)
- **IXSCAN**: busqueda por indice
- **FETCH**: recuperacion del documento desde el disco
- **SORT**: ordenacion en memoria

El tiempo de ejecucion, el numero de documentos examinados y los documentos devueltos se muestran en cada nodo del arbol de ejecucion. Si `docsExamined` es muy superior a `nReturned`, la consulta necesita un indice compuesto optimizado.

## Database Tools

El paquete `mongodb-database-tools` agrupa utilidades de linea de comandos para exportacion, importacion y monitoreo. Se instalan por separado del servidor:

```bash
# Ubuntu/Debian
sudo apt-get install -y mongodb-database-tools

# macOS
brew install mongodb-database-tools
```

### mongodump y mongorestore

`mongodump` crea una copia binaria de una base de datos o coleccion en formato BSON. Esta copia incluye tanto los datos como los metadatos (indices, colecciones).

```bash
# Backup completo de la base de datos mi_bd
mongodump --uri="mongodb://localhost:27017" --db=mi_bd --out=/backups/mi_bd_$(date +%Y%m%d)
```

El directorio resultante contiene un archivo `.bson` por cada coleccion y un archivo `metadata.json` con los indices.

`mongorestore` restaura una copia creada con `mongodump`:

```bash
# Restaurar desde el directorio de backup
mongorestore --uri="mongodb://localhost:27017/mi_bd" /backups/mi_bd_20240101/mi_bd
```

Opciones importantes:
- `--drop`: elimina la coleccion antes de restaurar (evita duplicados)
- `--nsInclude`: restaura solo colecciones especificas
- `--gzip`: comprime/descomprime los archivos BSON en vuelo

> `mongodump` no es un sustituto de backup en caliente con consistencia transaccional. Para replica sets, se debe ejecutar contra un secundario usando `--readPreference=secondary` o, preferiblemente, usar snapshots del sistema de archivos.

### mongoexport y mongoimport

A diferencia de `mongodump`, que trabaja en formato BSON binario, `mongoexport` produce JSON o CSV legible:

```bash
# Exportar a JSON
mongoexport --uri="mongodb://localhost:27017/mi_bd" \
    --collection=usuarios \
    --out=usuarios.json

# Exportar a CSV con campos especificos
mongoexport --uri="mongodb://localhost:27017/mi_bd" \
    --collection=usuarios \
    --fields=nombre,email,edad \
    --type=csv \
    --out=usuarios.csv
```

`mongoimport` realiza la operacion inversa:

```bash
# Importar desde JSON
mongoimport --uri="mongodb://localhost:27017/mi_bd" \
    --collection=usuarios \
    --file=usuarios.json

# Importar desde CSV
mongoimport --uri="mongodb://localhost:27017/mi_bd" \
    --collection=usuarios \
    --type=csv \
    --headerline \
    --file=usuarios.csv
```

La opcion `--headerline` interpreta la primera linea del CSV como nombres de campos. `--mode` controla el comportamiento ante documentos existentes: `insert` (falla si existe), `upsert` (actualiza si existe), `merge` (fusiona campos), `replace` (reemplaza completamente).

### mongostat

`mongostat` muestra metricas en tiempo real de la actividad del servidor, similar a `vmstat` o `iostat` de Linux:

```bash
mongostat --uri="mongodb://localhost:27017" --rowcount=5 1
```

```text
insert query update delete getmore command dirty used
   123   456     78     12       3     567   0.1  82.3
```

Las columnas principales:
- **insert/query/update/delete**: operaciones por segundo
- **getmore**: veces que el cursor pide mas lotes de resultados
- **command**: comandos totales por segundo (incluye internos)
- **dirty**: porcentaje de paginas sucias en cache
- **used**: porcentaje de cache utilizado

Un valor alto en `dirty` sostenido en el tiempo indica que la escritura supera la capacidad de descarga a disco. Un valor alto en `used` sin liberacion sugiere que el cache es insuficiente para el working set.

### mongotop

`mongotop` monitoriza el tiempo que `mongod` pasa en operaciones de lectura y escritura por coleccion:

```bash
mongotop 2
```

```text
ns                        total      read     write
mi_bd.usuarios            15ms        2ms      13ms
mi_bd.pedidos             42ms       40ms       2ms
admin.system.users         0ms        0ms       0ms
```

La columna `total` es la suma de `read` + `write` en el intervalo. Si una coleccion muestra consistentemente valores altos en `write`, puede necesitar un indice mas eficiente o un shard key mejor distribuida.

> `mongotop` mide el tiempo de CPU dentro de `mongod`, no la latencia de red o disco. Un tiempo alto en `read` sin consultas aparentes puede indicar operaciones internas como la aplicacion del journal o la compactacion de datos.

## Integracion en scripts

Todas las Database Tools soportan salida silenciosa y codigos de retorno, lo que permite integrarlas en pipelines de automatizacion:

```bash
#!/bin/bash
# backup_diario.sh
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
mongodump --uri="$MONGO_URI" --out="/backups/$TIMESTAMP" --quiet
if [ $? -eq 0 ]; then
    echo "Backup completado: $TIMESTAMP"
else
    echo "Error en backup: $TIMESTAMP" >&2
    exit 1
fi
```

La combinacion de `--quiet` con la verificacion del codigo de retorno (`$?`) permite construir pipelines robustos sin depender de la salida textual de las herramientas.

