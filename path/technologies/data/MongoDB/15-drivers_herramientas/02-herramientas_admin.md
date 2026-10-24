# Herramientas de administracion

El ecosistema de MongoDB incluye un conjunto de herramientas que cubren desde la administracion diaria via terminal hasta la inspeccion visual de esquemas y el analisis de rendimiento. Estas herramientas se distribuyen en tres grupos: el shell moderno (mongosh), la interfaz grafica (Compass) y el paquete de utilidades de base de datos (Database Tools). Conocer su funcionamiento y saber interpretar su salida es tan importante como dominar el lenguaje de consultas, porque son el medio a traves del cual se diagnostican problemas de rendimiento, se realizan backups y se audita el estado del cluster.

## mongosh

mongosh es el shell de MongoDB moderno que reemplazo al antiguo `mongo` en la version 5.0. Esta escrito en Node.js y proporciona autocompletado con tabulador, resaltado de sintaxis, historial de comandos, y soporte para scripts JavaScript con ES6+. Se conecta a una instancia mongod, a un replica set o a un cluster shardeado mediante la misma cadena de conexion que usan los drivers.

```bash
# Conexion basica
mongosh "mongodb://localhost:27017"

# Conexion con autenticacion
mongosh "mongodb://user:pass@localhost:27017/admin"

# Conexion a Atlas
mongosh "mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/tienda"

# Ejecutar script directamente
mongosh script.js

# Evaluar expresion
mongosh --eval "db.version()"
```

Una vez dentro del shell, el flujo de trabajo tipico combina metacomandos (`use`, `show`, `exit`) con operaciones sobre colecciones:

```javascript
use tienda
show collections
db.productos.find({ precio: { $gt: 500 } }).pretty()
```

mongosh expone el objeto `Mongo()` para manejar multiples conexiones y el metodo `load()` para cargar scripts reutilizables. Ademas, proporciona helpers para administracion como `rs.status()`, `sh.status()`, `db.currentOp()` y `db.serverStatus()`.

## MongoDB Compass

Compass es la interfaz grafica oficial de MongoDB. Ofrece un explorador visual de datos, un constructor de consultas con autocompletado, un analizador de esquemas (schema analysis) y herramientas de rendimiento. Esta disponible como aplicacion de escritorio independiente o integrada en MongoDB Atlas.

**Exploracion de datos:** permite navegar por las bases de datos y colecciones, visualizar documentos en formato JSON o tabla, y aplicar filtros graficamente. El query bar acepta sintaxis nativa de MongoDB con autocompletado.

**Schema analysis:** analiza una coleccion y genera un resumen visual de los campos, sus tipos, valores mas frecuentes, rangos y distribucion. Es util para entender la estructura de una coleccion sin esquema fijo y detectar anomalias como campos que deberian ser numeros pero aparecen como strings.

```json
// Ejemplo de lo que schema analysis revela
{
  "campo": "edad",
  "tipos": { "int": 85, "double": 10, "string": 5 },
  "valores_frecuentes": [25, 30, 35],
  "rango": { "min": 18, "max": 85 }
}
```

**Performance insights:** muestra las operaciones lentas, el uso de indices, el throughput de lecturas/escrituras y el tiempo de respuesta del servidor. Se integra con el profiler de MongoDB para identificar consultas que realizan COLLSCAN o que tienen tiempos de ejecucion elevados.

**Agregation pipeline builder:** proporciona un editor visual para construir pipelines de agregacion paso a paso, anadiendo etapas desde un panel y viendo los resultados intermedios en tiempo real.

## MongoDB Database Tools

El paquete `mongodb-database-tools` incluye utilidades de linea de comandos para exportacion, importacion, monitoreo y reproduccion de trafico. Se instalan por separado del servidor y estan disponibles para todos los sistemas operativos.

### mongodump y mongorestore

mongodump realiza un backup logico de la base de datos en formato BSON. Es la herramienta mas utilizada para backups completos, aunque no es adecuada para conjuntos de datos que superan varios cientos de gigabytes (ahi es preferible usar snapshots de disco o Atlas backups).

```bash
# Backup completo de una base de datos
mongodump --db tienda --out /backups/2024-01-15/

# Backup con oplog para point-in-time recovery
mongodump --db tienda --oplog --out /backups/tienda_oplog/

# Backup de una sola coleccion
mongodump --db tienda --collection productos --out /backups/

# Backup de todas las bases de datos
mongodump --out /backups/full/
```

mongorestore restaura desde el formato BSON de mongodump. Puede restaurar una base de datos completa, una coleccion especifica, o reemplazar documentos existentes.

```bash
# Restaurar base de datos completa
mongorestore --db tienda /backups/2024-01-15/tienda/

# Restaurar con replays de oplog para point-in-time
mongorestore --oplogReplay /backups/tienda_oplog/

# Restaurar solo una coleccion
mongorestore --db tienda --collection productos /backups/tienda/productos.bson

# Restaurar sin sobrescribir documentos existentes
mongorestore --db tienda --drop /backups/tienda/
```

### mongoexport y mongoimport

mongoexport exporta datos a JSON o CSV, util para intercambiar datos con otros sistemas. No preserva los tipos BSON (ObjectId, Date, BinData se convierten a strings o formatos estandar).

```bash
# Exportar a JSON
mongoexport --db tienda --collection productos --out productos.json

# Exportar a CSV con campos especificos
mongoexport --db tienda --collection productos --fields nombre,precio,categoria --type=csv --out productos.csv

# Exportar con query
mongoexport --db tienda --collection productos --query '{ "precio": { "$gt": 500 } }' --out caros.json
```

mongoimport realiza la operacion inversa: importa desde JSON o CSV.

```bash
mongoimport --db tienda --collection productos --file productos.json
mongoimport --db tienda --collection productos --type=csv --headerline --file productos.csv
```

### mongostat

mongostat muestra metricas en tiempo real del servidor MongoDB, similares a `vmstat` o `iostat` en Linux. Se conecta a la instancia y actualiza las estadisticas cada segundo (por defecto).

```bash
mongostat --host localhost:27017

# Ejemplo de salida:
# insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn time
#    10    50     20      5       2     100   2.3   4.1       2  2.1G 512M 0|0  0|0   20k     5k   15 09:15:01
```

Las columnas principales:
- `insert/query/update/delete`: operaciones por segundo.
- `getmore`: cursores que piden mas datos.
- `command`: comandos totales.
- `dirty/used`: porcentaje del cache WiredTiger sucio y usado.
- `flushes`: escrituras a disco por segundo.
- `vsize/res`: memoria virtual y residente.
- `qrw`: cola de lecturas/escrituras (lectores|escritores).
- `arw`: lectores/escritores activos.
- `net_in/net_out`: trafico de red.

> Una cola de escrituras (qw) consistentemente alta indica que el servidor no da abasto con las escrituras. Si ademas el porcentaje dirty es alto, el cache de WiredTiger esta saturado.

### mongotop

mongotop muestra el tiempo que pasa el servidor realizando operaciones de lectura y escritura en cada coleccion. Es util para identificar colecciones con alta contention.

```bash
mongotop --host localhost:27017

# Ejemplo de salida:
#                     ns    total    read   write    2024-01-15T09:15:01Z
#  tienda.productos          15ms    10ms     5ms
#  tienda.pedidos            42ms     2ms    40ms
#  admin.system.roles         0ms     0ms     0ms
```

En el ejemplo, la coleccion `pedidos` tiene 40ms de escritura, indicando que es un punto caliente. Combinado con `mongostat`, permite correlacionar el volumen de operaciones con el tiempo invertido por coleccion.

### mongoreplay

mongoreplay captura, registra y reproduce trafico de MongoDB. Se usa para pruebas de carga: se captura el trafico de produccion y se reproduce en un entorno de staging o testing.

```bash
# Capturar trafico
mongoreplay record -i eth0 -p 27017 -o captura/

# Reproducir trafico
mongoreplay play -p 27018 -f captura/
```

mongoreplay es util para validar cambios de indice, actualizaciones de version o cambios de configuracion reproduciendo la carga real de produccion.

