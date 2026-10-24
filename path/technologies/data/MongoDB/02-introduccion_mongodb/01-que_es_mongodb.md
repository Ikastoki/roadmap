# Que es MongoDB

MongoDB irrumpio en el panorama de bases de datos en 2009 de la mano de 10gen (hoy MongoDB Inc.) como respuesta a las limitaciones que los sistemas relacionales presentaban frente a cargas de trabajo masivas, esquemas cambiantes y requisitos de escalabilidad horizontal. Casi una decada y media despues, se ha consolidado como el principal representante de las bases de datos documentales, combinando la flexibilidad de un modelo sin esquema fijo con prestaciones tradicionalmente reservadas a los motores SQL: transacciones ACID multi-documento, indices secundarios de alto rendimiento y un potente pipeline de agregacion.

La historia de MongoDB es tambien la historia de su motor de almacenamiento. Hasta la version 3.0 convivian dos motores: MMAPv1 (el original, basado en mapeo de memoria) y WiredTiger, adquirido junto con la compania homonima. A partir de MongoDB 3.2, WiredTiger se convirtio en el motor por defecto, y desde 4.2 es el unico disponible. Esta transicion supuso un salto cualitativo en compresion, concurrencia y eficiencia de cache. La licencia del proyecto tambien evoluciono: de AGPL v3 paso a SSPL (Server Side Public License) en 2018, lo que genero debate en la comunidad open source. En la actualidad, MongoDB 7.x representa la madurez del producto, con mejoras en rendimiento, seguridad y escalabilidad.

## Caracteristicas fundamentales

MongoDB almacena los datos en documentos BSON (Binary JSON), una representacion binaria de JSON que extiende el conjunto de tipos soportados: ObjectId, Date, Decimal128, BinData, entre otros. A diferencia de una base de datos relacional, no exige un esquema predefinido: cada documento de una misma coleccion puede tener campos distintos. Esta flexibilidad resulta especialmente util en escenarios donde la estructura de los datos evoluciona con rapidez o donde distintos tipos de entidad comparten el mismo contenedor.

Sobre este modelo de documentos, MongoDB ofrece un sistema de indices que incluye indices simples, compuestos, multikey (para arrays), text (busqueda de texto completo), geoespaciales, TTL (expiran automaticamente), parciales, sparse y hashed. Cada indice se implementa como un arbol B+, y el planificador de consultas selecciona automaticamente el indice mas eficiente.

El aggregation framework constituye uno de los puntos fuertes del producto. Se trata de un pipeline de procesamiento de datos compuesto por etapas (`$match`, `$group`, `$project`, `$sort`, `$lookup`, `$facet`, etc.) que permite transformar y analizar documentos con una expresividad comparable a la de SQL, pero operando directamente sobre el modelo documental.

En el plano operacional, MongoDB integra de forma nativa la replicacion mediante replica sets (conjuntos de replicas) y la particion horizontal mediante sharding. Un replica set proporciona alta disponibilidad con failover automatico; el sharding distribuye los datos entre multiples servidores, permitiendo escalar mas alla de los limites de una maquina individual.

Desde la version 4.0, MongoDB soporta transacciones multi-documento con garantias ACID sobre replica sets, y desde 4.2 tambien en entornos fragmentados (sharded clusters). Aunque el modelo documental reduce la necesidad de transacciones distribuidas (gracias a los documentos embebidos), cuando se requieren operaciones atomicas que abarcan varios documentos, MongoDB ofrece las mismas garantias que un motor relacional.

Los change streams completan el ecosistema: un mecanismo de suscripcion en tiempo real a los cambios que ocurren en una coleccion, base de datos o todo el cluster. Basados en el oplog, permiten reaccionar a inserciones, actualizaciones y eliminaciones, lo que los hace ideales para sincronizacion de datos, notificaciones en vivo o pipelines de eventos.

## Motor WiredTiger

WiredTiger es el motor de almacenamiento exclusivo de MongoDB desde la version 4.2. Su diseno se basa en tres pilares: compresion avanzada de datos, control de concurrencia multiversion (MVCC) optimista y una gestion de cache sofisticada.

La compresion opera a nivel de pagina y se aplica tanto a los datos como a los indices. WiredTiger soporta tres algoritmos: snappy (rapido, compresion moderada, activado por defecto), zlib (mayor compresion, mas coste CPU) y zstd (buen equilibrio entre velocidad y tasa de compresion). La eleccion del algoritmo depende del perfil de la carga de trabajo: snappy minimiza la latencia, mientras que zlib o zstd reducen el espacio en disco a costa de mayor uso de CPU.

WiredTiger emplea un cache de documentos en memoria cuya configuracion por defecto es el 50 % de la RAM disponible (menos 1 GB). Este cache almacena tanto los datos como los indices mas utilizados, y su gestion determina en gran medida el rendimiento general del servidor. Cuando la cache se satura, WiredTiger aplica un algoritmo de reemplazo basado en frecuencia de acceso (no simplemente LRU) y pagina los datos menos usados hacia el disco en unidades de 4 KB.

El control de concurrencia se realiza mediante snapshots a nivel de documento. Cada operacion de lectura ve una snapshot consistente de los datos en un instante determinado, mientras que las escrituras adquieren bloqueos a nivel de documento. Este enfoque optimista minimiza la contention y permite que lecturas y escrituras avancen en paralelo sin bloquearse mutuamente, siempre que no accedan al mismo documento.

El journaling de WiredTiger garantiza la durabilidad de las escrituras incluso ante un fallo abrupto del sistema. Cada operacion de escritura se registra primero en el journal (almacenado en el directorio `journal/`) y posteriormente se aplica a los archivos de datos. MongoDB permite configurar el intervalo de confirmacion del journal (por defecto 100 ms) y si el cliente debe esperar a la confirmacion antes de recibir el acknowledge.

## Casos de uso

MongoDB destaca en escenarios donde la flexibilidad del esquema, la escalabilidad horizontal y el rendimiento en lecturas y escrituras son criticos.

- **Aplicaciones web y moviles**: perfiles de usuario, sesiones, notificaciones y contenido personalizado. La integracion con JSON y la disponibilidad de drivers para practicamente todos los lenguajes facilitan el desarrollo.
- **Catalogos de productos**: la variabilidad de atributos entre categorias (una camiseta tiene talla y color; un televisor tiene pulgadas y resolucion) encaja naturalmente con el modelo de documentos.
- **Sistemas de gestion de contenido (CMS)**: articulos, comentarios, metadatos y versiones se almacenan en documentos que reflejan directamente la estructura del contenido.
- **Internet de las cosas (IoT)**: ingesta masiva de lecturas de sensores con el bucket pattern para agrupar datos temporales y TTL indices para expirar automaticamente los datos antiguos.
- **Videojuegos (gaming)**: inventarios de jugadores, leaderboards, estados de partida y eventos en tiempo real se benefician de la baja latencia y el esquema flexible.
- **Analitica y big data**: el aggregation framework permite ejecutar pipelines complejos de analisis directamente sobre los datos de operacion, sin necesidad de un sistema ETL separado.

