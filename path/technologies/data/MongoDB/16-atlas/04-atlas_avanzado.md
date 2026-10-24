# Atlas avanzado

Mas alla del alojamiento gestionado de bases de datos, Atlas integra un ecosistema de servicios que extienden las capacidades de MongoDB a casos de uso como la sincronizacion offline-first para dispositivos moviles, la consulta federada sobre multiples fuentes de datos, la visualizacion de datos mediante dashboards y la distribucion global de datos con baja latencia. Estos servicios comparten la misma infraestructura de Atlas y se gestionan desde la misma consola, lo que elimina la complejidad de integrar productos heterogeneos.

## Atlas Device SDK (Realm)

Atlas Device SDK, anteriormente conocido como MongoDB Realm, es un conjunto de herramientas para construir aplicaciones offline-first en dispositivos moviles, web y escritorio. Su componente central es Realm, una base de datos local embebida que almacena los datos en el dispositivo y los sincroniza con Atlas en segundo plano cuando hay conectividad.

### Base de datos local Realm

Realm no es una base de datos SQL ni una abstraccion sobre SQLite. Es un motor de almacenamiento orientado a objetos construido desde cero para dispositivos moviles, que almacena los datos directamente como objetos en un formato binario propio. Cada objeto Realm se corresponde con una clase del lenguaje de la aplicacion (Swift, Kotlin, JavaScript, C#), y las relaciones entre objetos se resuelven mediante referencias directas sin joins.

La base de datos local es reactiva: cualquier cambio en los datos dispara notificaciones automaticas a las partes de la aplicacion que estan observando esos datos. Esto permite construir interfaces que se actualizan en tiempo real sin polling ni consultas repetitivas.

```dart
// Ejemplo con Realm SDK para Flutter
class Producto extends Object {
  @PrimaryKey()
  late ObjectId id;

  late String nombre;
  late double precio;
  late bool disponible;
}

final producto = Producto()
  ..id = ObjectId()
  ..nombre = "Teclado mecanico"
  ..precio = 89.99
  ..disponible = true;

realm.write(() => realm.add(producto));
```

### Sincronizacion Device Sync

Device Sync es el sistema de sincronizacion bidireccional entre la base de datos local Realm y el cluster de Atlas. Cuando el dispositivo esta online, los cambios locales se replican a Atlas y los cambios remotos se descargan al dispositivo, todo ello de forma transparente para la aplicacion.

La sincronizacion se basa en un modelo de suscripcion: la aplicacion define que datos quiere sincronizar mediante consultas basadas en MongoDB. Por ejemplo, una aplicacion de tareas puede suscribirse a las tareas del usuario actual:

```javascript
realm.subscriptions.update((mutableSubscriptions) => {
  mutableSubscriptions.add(
    realm.objects('Tarea').filtered('usuario_id == $0', usuarioId)
  );
});
```

Solo los datos que coinciden con las suscripciones se almacenan localmente. Cuando el usuario anyade una tarea en offline, Realm la persiste localmente y, al recuperar la conectividad, la sincroniza con Atlas. Si otro usuario modifica la misma tarea desde otro dispositivo, Realm resuelve los conflictos mediante un algoritmo de ultimo escritor (LWW) o conflict resolution personalizado.

La sincronizacion offline-first es especialmente util para aplicaciones que funcionan en entornos con conectividad intermitente, como entregas a domicilio, inspecciones de campo o aplicaciones de venta en tiendas fisicas. El desarrollador escribe la logica de negocio asumiendo que los datos estan siempre disponibles localmente, y Realm se encarga de la sincronizacion.

### Autenticacion y funciones serverless

Atlas Device SDK incluye un sistema de autenticacion que soporta proveedores como email/password, inicio de sesion con Google, Apple, Facebook, JWT personalizado y autenticacion anonima. La autenticacion se integra con las reglas de permisos que controlan que datos puede leer o escribir cada usuario.

Las funciones serverless de Atlas (antes Realm Functions) permiten ejecutar codigo JavaScript del lado del servidor en respuesta a eventos como la creacion de un documento, una peticion HTTPS entrante o una llamada desde la aplicacion movil. Estas funciones se ejecutan en el entorno de Atlas y pueden acceder a la base de datos, a servicios HTTP externos y a los servicios de autenticacion. Son equivalentes a las cloud functions de AWS Lambda o Google Cloud Functions pero integradas directamente con el ecosistema de Atlas.

```javascript
// Funcion serverless que se ejecuta al crear un pedido
exports = function(changeEvent) {
  const { fullDocument } = changeEvent;
  const client = context.services.get('mongodb-atlas');
  const productos = client.db('tienda').collection('productos');

  fullDocument.items.forEach(item => {
    productos.updateOne(
      { _id: item.producto_id },
      { $inc: { stock: -item.cantidad } }
    );
  });
};
```

## Data Federation

Data Federation es un motor de consultas que permite ejecutar operaciones de MongoDB sobre datos almacenados fuera de Atlas, como si fueran colecciones nativas del cluster. Utiliza el mismo aggregation pipeline, los mismos drivers y las mismas herramientas de MongoDB, pero los datos se leen directamente desde el almacenamiento externo sin necesidad de importarlos previamente.

Las fuentes de datos soportadas incluyen:

- **Almacenamiento en la nube**: AWS S3, Azure Blob Storage, Google Cloud Storage.
- **HTTP/HTTPS**: datasets accesibles mediante URL.
- **Atlas clusters**: otros clusters de Atlas dentro de la misma organizacion.
- **Online Archive**: datos archivados automaticamente desde Atlas a S3 cuando superan una antiguedad configurada.

La configuracion de Data Federation se realiza mediante stores y virtual collections. Un store define la fuente de datos externa (tipo, bucket, region, credenciales). Una virtual collection mapea archivos de ese store (JSON, BSON, CSV, Parquet, Avro) a una coleccion virtual que se consulta con sintaxis MongoDB estandar.

La principal utilidad de Data Federation es el tiering de datos: los datos calientes se mantienen en el cluster de Atlas para bajo latencia, mientras que los datos frios (logs historicos, registros de auditoria, datos de mas de 90 dias) se archivan automaticamente en S3 con Online Archive. Las consultas que acceden tanto a datos calientes como frios se ejecutan de forma transparente, combinando resultados del cluster y del almacenamiento externo en una sola respuesta.

```javascript
// Consulta que combina datos del cluster con datos federados
db.ventas.aggregate([
  { $match: { fecha: { $gte: ISODate("2023-01-01") } } },
  { $group: { _id: "$region", total: { $sum: "$importe" } } },
  { $sort: { total: -1 } }
]);
// Esta consulta accede a ventas_recientes (cluster) y ventas_historicas (S3) unificadas
```

Data Federation no soporta escrituras: es exclusivamente un motor de consulta. Los datos solo pueden modificarse en su origen (el bucket S3 o el cluster Atlas original).

## Atlas Charts

Atlas Charts es un servicio de visualizacion de datos integrado en Atlas que permite crear graficos y dashboards a partir de los datos almacenados en el cluster, sin necesidad de herramientas de BI externas ni de exportar los datos.

Los graficos se construyen desde el editor visual de Charts, donde se seleccionan las colecciones, se definen los campos a visualizar y se configura el tipo de grafico. Charts ejecuta las consultas de agregacion en segundo plano y representa los resultados en graficos de barras, lineas, areas, circulares, tablas pivotantes, mapas de calor, mapas geoespaciales y tablas de datos.

Las caracteristicas principales de Atlas Charts incluyen:

- **Conexion directa**: los graficos consultan los datos en tiempo real desde el cluster Atlas. No hay pipelines ETL ni almacenamiento intermedio.
- **Filtros interactivos**: los dashboards permiten al usuario final filtrar los datos por rango de fechas, categorias o cualquier campo, y los graficos se actualizan automaticamente.
- **Agregaciones personalizadas**: se puede definir un pipeline de agregacion completo como fuente de datos del grafico, permitiendo transformaciones complejas antes de la visualizacion.
- **Embedding**: los graficos y dashboards pueden incrustarse en aplicaciones web mediante un iframe o la API de Charts, con autenticacion integrada.
- **Exportacion**: los graficos se exportan como imagenes PNG o datos CSV para su inclusion en informes.

Los dashboards se organizan en paneles que agrupan varios graficos relacionados. Cada dashboard tiene permisos de lectura configurables por usuario o por rol, permitiendo compartir visualizaciones con el equipo o con clientes sin necesidad de que tengan acceso directo a la base de datos.

Atlas Charts esta disponible en todos los planes de Atlas, incluido el tier gratuito M0, aunque con limitaciones en el numero de graficos y dashboards.

## Global clusters

Los global clusters resuelven el problema de la latencia de lectura para aplicaciones con usuarios distribuidos geograficamente. Un cluster Atlas estandar tiene todos sus nodos en una sola region cloud. Si un usuario en Europa intenta leer datos de un cluster en us-east-1 (Virginia), cada consulta recorre aproximadamente 6000 km, anadiendo entre 80 y 120 ms de latencia solo por la distancia geografica.

Un global cluster distribuye sus nodos de lectura en multiples regiones, manteniendo un unico primario en la region de escritura. Los usuarios se conectan a la replica mas cercana mediante la preferencia de lectura `nearest`, que el driver resuelve consultando la latencia de red a cada nodo o utilizando las etiquetas de zona configuradas.

La configuracion de un global cluster implica:

1. Crear un cluster con multiples regiones en la consola de Atlas.
2. Definir las etiquetas de zona (zone tags) que asocian rangos de la shard key con regiones concretas.
3. Configurar la preferencia de lectura en la aplicacion para utilizar `nearest`.

```javascript
const client = new MongoClient(uri, {
  readPreference: 'nearest',
  readConcern: { level: 'local' }
});
```

El sharding es un requisito para los global clusters, ya que la distribucion de datos por region se realiza mediante las reglas de zona del sharding. La shard key debe elegirse de modo que los datos que pertenecen a una region se almacenen en los nodos de esa region. Por ejemplo, una aplicacion con usuarios en Europa y Asia puede usar `region` como parte de la shard key:

```javascript
sh.shardCollection("app.datos", { "region": 1, "_id": 1 });
sh.addShardTag("shard-europa", "europa");
sh.addShardTag("shard-asia", "asia");
sh.updateZoneKeyRange("app.datos", { "region": "EU" }, { "region": "EU\0" }, "europa");
sh.updateZoneKeyRange("app.datos", { "region": "AS" }, { "region": "AS\0" }, "asia");
```

Con esta configuracion, los documentos con `region: "EU"` se almacenan en los shards etiquetados como `europa`, y los usuarios europeos leen desde esos shards con baja latencia. Las escrituras, sin embargo, siempre se dirigen al primario del cluster, que puede estar en cualquier region, introduciendo latencia de escritura para las regiones remotas.

Para aplicaciones que requieren baja latencia tanto en lecturas como en escrituras en multiples regiones, Atlas ofrece la topologia multi-region gestionada, donde el primario puede moverse entre regiones o configurarse un cluster con replicas de escritura distribuidas, aunque esta configuracion tiene implicaciones de consistencia que deben evaluarse cuidadosamente.

