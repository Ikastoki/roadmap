# Arquitectura de MongoDB

La arquitectura de MongoDB se organiza en torno a unos pocos componentes conceptuales y de proceso que, comprendidos en conjunto, permiten entender como fluyen los datos desde que el cliente emite una consulta hasta que el motor de almacenamiento los persiste en disco.

## mongod: el proceso servidor

`mongod` es el demonio principal de MongoDB. Es el proceso que gestiona las conexiones entrantes, ejecuta las consultas, mantiene los indices, coordina la replicacion y el sharding, y se comunica con el motor de almacenamiento para leer y escribir datos en disco.

Por defecto, `mongod` escucha en el puerto **27017** y acepta conexiones en la interfaz configurada mediante la directiva `net.bindIp`. En un despliegue tipico se ejecuta como un servicio del sistema (`systemctl start mongod`), pero tambien puede lanzarse manualmente para entornos de desarrollo o pruebas.

Cada instancia `mongod` puede desempenar distintos roles segun la configuracion del despliegue:

- **Servidor independiente (standalone)**: una unica instancia, util para desarrollo o entornos donde la alta disponibilidad no es critica.
- **Miembro de un replica set**: forma parte de un conjunto de replicas como primario (recibe escrituras) o secundario (replica los cambios).
- **Config server**: almacena los metadatos del cluster fragmentado (sharding).
- **Shard**: almacena un subconjunto de los datos en un cluster fragmentado.

Independientemente del rol, todos los `mongod` comparten la misma estructura interna: un capa de red que gestiona las conexiones (con pool de hilos), un planificador de consultas que elige los indices optimos, un motor de ejecucion que recorre los documentos y aplica filtros y proyecciones, y el motor de almacenamiento WiredTiger que se encarga de la persistencia.

## mongosh: el cliente por linea de comandos

`mongosh` es el shell interactivo de MongoDB, sucesor del antiguo shell `mongo`. Proporciona un entorno JavaScript completo (basado en Node.js) con autocompletado, resaltado de sintaxis y soporte para scripts. Se conecta a una instancia `mongod` (o a un `mongos` en entornos fragmentados) y permite ejecutar operaciones de administracion, consultas y agregaciones.

La conexion se realiza especificando la cadena de conexion:

```bash
mongosh "mongodb://localhost:27017"
```

Una vez conectado, `mongosh` expone el objeto global `db` que referencia la base de datos activa. Sobre el se invocan los metodos de las colecciones:

```javascript
db.usuarios.insertOne({ nombre: "Ana", edad: 34 })
db.usuarios.find({ edad: { $gt: 30 } })
```

`mongosh` incluye metacomandos como `use <bd>`, `show dbs`, `show collections` y `exit`, ademas de soportar la ejecucion de archivos JavaScript completos con `load()`.

## Documento BSON

El documento es la unidad fundamental de datos en MongoDB. Equivale a una fila en una base de datos relacional, pero con una diferencia esencial: no tiene un esquema fijo. Un documento es una estructura de pares campo-valor representada internamente en formato BSON.

BSON (Binary JSON) es una serializacion binaria de documentos similares a JSON que extiende el conjunto de tipos soportados:

| Tipo BSON | Descripcion |
|-----------|-------------|
| `ObjectId` | Identificador unico de 12 bytes |
| `String` | Cadena UTF-8 |
| `Int32`, `Int64` | Enteros de 32 y 64 bits |
| `Double` | Coma flotante IEEE 754 |
| `Decimal128` | Decimal de alta precision |
| `Boolean` | true / false |
| `Date` | Fecha/hora (milisegundos desde epoch) |
| `Binary` | Datos binarios arbitrarios |
| `Array` | Lista ordenada de valores |
| `Object` | Documento anidado (subdocumento) |
| `Null` | Valor nulo |
| `Timestamp` | Marca temporal interna |

El limite de tamano de un documento es de **16 MB**. Esta restriccion existe para garantizar que las operaciones de transferencia por red y el procesamiento en memoria sean manejables. Cuando un documento supera este limite, se debe reconsiderar el diseno (por ejemplo, separando los datos en multiples documentos y usando referencias).

Cada documento contiene un campo `_id` obligatorio que actua como clave primaria. Si el cliente no lo proporciona, MongoDB genera automaticamente un `ObjectId`.

## Coleccion

Una coleccion es un contenedor de documentos, analogo a una tabla en una base de datos relacional. La diferencia fundamental es que los documentos dentro de una misma coleccion no estan obligados a compartir la misma estructura: cada documento puede tener campos distintos.

Las colecciones se crean implicitamente al insertar el primer documento o explicitamente mediante `db.createCollection()`. Existen variantes especializadas:

- **Capped collections**: colecciones de tamano fijo que sobrescriben los documentos mas antiguos cuando alcanzan el limite. Utiles para logs y datos temporales.
- **Time-series collections**: optimizadas para datos temporales, almacenan los datos de forma columnar interna.
- **Collections con validacion**: se puede definir un esquema de validacion mediante `$jsonSchema` para restringir la estructura de los documentos admitidos.

Las colecciones tienen un namespace completo que incluye la base de datos y el nombre separados por punto: `basededatos.coleccion`.

## Base de datos

Una base de datos es el contenedor logico de colecciones. Proporciona un ámbito para la gestion de permisos (los usuarios se asignan a bases de datos) y para el aislamiento de datos. MongoDB crea implicitamente una base de datos cuando se inserta el primer documento en una coleccion dentro de ella.

En un despliegue tipico coexisten varias bases de datos con propositos distintos:

- **admin**: almacena usuarios administrativos y roles. Si la autenticacion esta habilitada, los usuarios deben autenticarse contra esta base de datos.
- **config**: en clusters fragmentados, contiene los metadatos del sharding.
- **local**: almacena datos especificos de la instancia local (oplog del replica set, por ejemplo). No se replica.

El acceso a las colecciones sigue la notacion `db.nombreColeccion`, donde `db` es el objeto que referencia la base de datos activa en el contexto del shell.

## WiredTiger en la arquitectura

WiredTiger es la capa inferior de la arquitectura de MongoDB. Actua como interfaz entre el motor de consultas y el almacenamiento fisico en disco. Cuando `mongod` necesita leer un documento, el motor de consultas solicita la pagina correspondiente a WiredTiger, que la busca primero en su cache en memoria y, si no esta, la lee del disco, la descomprime y la coloca en cache. En las escrituras, WiredTiger aplica los cambios en memoria, registra la operacion en el journal y, eventualmente, escribe las paginas modificadas en los archivos de datos.

Los archivos de datos de WiredTiger se almacenan en el directorio especificado por `storage.dbPath` (por defecto `/data/db`) con extension `.wt`. El journal reside en el subdirectorio `journal/`. Esta separacion permite estrategias de backup distintas para datos y journal.

