# MongoDB Atlas

MongoDB Atlas es la plataforma Database-as-a-Service (DBaaS) oficial de MongoDB Inc. que abstrae por completo la gestion de infraestructura, permitiendo desplegar, operar y escalar bases de datos MongoDB sin intervencion manual sobre servidores, sistemas operativos o motores de almacenamiento. A diferencia de una instalacion on-premise donde el equipo de operaciones debe configurar replica sets, gestionar backups, aplicar parches de seguridad y monitorizar la salud del cluster, Atlas encapsula todo ese trabajo en una capa de gestion automatizada. El resultado es que un equipo de desarrollo puede tener un cluster replicado y seguro en produccion en minutos, con caracteristicas que de otro modo requeririan ingenieria de infraestructura dedicada.

Atlas no es simplemente MongoDB alojado: es una plataforma que anade valor sobre el motor de base de datos con servicios integrados como busqueda full-text, sincronizacion de datos para dispositivos moviles, federacion de consultas sobre multiples fuentes y visualizacion de datos mediante dashboards. Todos estos servicios comparten la misma capa de seguridad, facturacion y gestion, lo que elimina la friccion de integrar productos de terceros.

## Database as a Service

El modelo DBaaS de Atlas elimina las tareas operativas que tradicionalmente acompanan a una base de datos en produccion. Cuando se crea un cluster, Atlas provisiona automaticamente los servidores, configura el replica set con el numero de nodos elegido, establece la replicacion entre zonas de disponibilidad y activa el monitoreo con métricas predeterminadas. El equipo solo necesita preocuparse por el esquema de datos y las consultas; todo lo demas (parches de seguridad, actualizaciones de version, ajuste de almacenamiento) lo gestiona la plataforma.

El aprovisionamiento se realiza desde la interfaz web de Atlas, la API REST o proveedores de infraestructura como Terraform. Atlas soporta los tres principales proveedores cloud (AWS, Azure y GCP) y permite elegir la region donde se despliegan los datos. Durante la creacion del cluster se selecciona el proveedor, la region, el tamano del nodo, el numero de replicas y la version de MongoDB. Una vez creado, Atlas orquesta la provision de los recursos y devuelve una cadena de conexion lista para usar.

El auto-scaling es una de las prestaciones mas valoradas. Atlas puede ajustar automaticamente el tamano del cluster en funcion de la carga de trabajo, tanto verticalmente (aumentar o reducir la capacidad de CPU/RAM de los nodos) como horizontalmente (anyadir o eliminar shards en un cluster fragmentado). Las politicas de escalado se configuran con umbrales de utilizacion de CPU, latencia de operaciones o tamano de los datos, y Atlas ejecuta los cambios sin ventanas de mantenimiento.

> El auto-scaling vertical en Atlas se basa en la metrica `CPU utilization`: si la CPU supera el 70 % durante un periodo continuado, Atlas escala el cluster al siguiente tamano disponible. De igual modo, si la CPU baja del 10 %, reduce el tamano para optimizar costes.

## Plan gratuito M0

Atlas ofrece un tier gratuito denominado M0 que permite probar MongoDB sin coste alguno. Este cluster compartido proporciona 512 MB de almacenamiento y capacidades de procesamiento limitadas, pero incluye caracteristicas que lo hacen util para desarrollo, testing y prototipado. El M0 soporta replica sets de tres nodos gestionados automaticamente, backups programados con retencion de 7 dias, monitorizacion basica con alertas, y la mayoria de las funcionalidades de MongoDB, incluyendo aggregation pipeline, indices de texto y transacciones.

El cluster M0 se ejecuta en infraestructura compartida: los recursos de CPU y memoria se distribuyen entre varios clusters en el mismo servidor. Esto implica que el rendimiento no es predecible bajo cargas sostenidas y que ciertas operaciones intensivas pueden verse estranguladas. Por esta razon, MongoDB recomienda no utilizar M0 en entornos de produccion. Sin embargo, para un equipo que esta aprendiendo MongoDB o desarrollando una aplicacion en fase temprana, el M0 ofrece un entorno funcional completo sin necesidad de proporcionar una tarjeta de credito.

La creacion de un cluster M0 es inmediata: Atlas lo aprovisiona en segundos y esta listo para recibir conexiones desde cualquier parte del mundo, siempre que la IP del cliente este en la whitelist.

## Planes de pago

Por encima del M0, Atlas ofrece una jerarquia de planes que se diferencian en capacidad de procesamiento, memoria, almacenamiento y caracteristicas disponibles.

Los planes M2 y M5 son clusters compartidos de pago que superan las limitaciones del M0: ofrecen hasta 5 GB de almacenamiento (M5) y un rendimiento superior al del tier gratuito, aunque siguen ejecutandose en infraestructura compartida. Son adecuados para aplicaciones en produccion con cargas de trabajo ligeras.

A partir de M10 comienzan los clusters dedicados. Cada nodo es una instancia cloud exclusiva para el cluster, sin contención de recursos con otros inquilinos. La serie M sigue esta progresion: M10, M20, M30, M40, M50, M60, M80, M100, M120, M140, M160, M200, M300, M400 y M700. Cada salto duplica aproximádamente la capacidad de CPU, RAM y almacenamiento del escalon anterior.

| Plan   | RAM  | Almacenamiento | Tipo        |
|--------|------|----------------|-------------|
| M0     | 512 MB compartido | 512 MB | Compartido  |
| M2     | 2 GB compartido   | 2 GB   | Compartido  |
| M5     | 5 GB compartido   | 5 GB   | Compartido  |
| M10    | 2 GB dedicado     | 10 GB  | Dedicado    |
| M20    | 4 GB dedicado     | 20 GB  | Dedicado    |
| M30    | 8 GB dedicado     | 40 GB  | Dedicado    |
| M40    | 16 GB dedicado    | 80 GB  | Dedicado    |
| M50    | 32 GB dedicado    | 160 GB | Dedicado    |
| M60    | 64 GB dedicado    | 320 GB | Dedicado    |

Los clusters dedicados soportan auto-scaling, backups continuos con point-in-time recovery, monitoring avanzado con alerts personalizables, cifrado en reposo gestionado por el cliente (CMK), VPC peering, private endpoints y auditoria. Los planes superiores (M60 en adelante) incluyen ademas soporte para sharding nativo, necesario para escalar horizontalmente cuando los datos superan la capacidad de un solo nodo.

Ademas del dimensionamiento vertical, Atlas ofrece la posibilidad de anyadir replicas de lectura adicionales en distintas regiones (global clusters) y de activar la fragmentacion (sharding) para distribuir los datos entre varios conjuntos de replicas.

> El almacenamiento en Atlas utiliza discos SSD respaldados por el proveedor cloud. En AWS se emplean gp2/gp3 (EBS), en Azure managed disks y en GCP persistent disks. El almacenamiento real disponible puede ser inferior al contratado debido a la sobrecarga del motor y los indices.

## Caracteristicas principales

### Clusters gestionados

Cada cluster en Atlas es un replica set de MongoDB completamente gestionado. Atlas configura automaticamente la replicacion entre nodos, distribuyendolos en distintas zonas de disponibilidad dentro de la misma region para tolerar fallos del centro de datos. Si el nodo primario falla, Atlas detecta la perdida de heartbeat y desencadena una eleccion automatica, promoviendo un secundario a primario en segundos. Todo el proceso es transparente para la aplicacion, que solo debe implementar logica de reconexion.

El numero de nodos por replica set es configurable: el minimo recomendado es 3 (un primario y dos secundarios), pero Atlas permite hasta 10 nodos por replica set, incluyendo nodos ocultos para backups y nodos con retraso (delayed members) para recuperacion ante errores logicos.

### Backups automatizados

Atlas ofrece tres modalidades de backup. Los backups programados (snapshots) capturan el estado completo del cluster a intervalos regulares: cada 6 horas con retencion de 2 dias, cada 24 horas con retencion de 7 dias, o semanalmente con retencion de 4 semanas, segun la configuracion. Los snapshots son almacenados en el mismo proveedor cloud y pueden restaurarse con un solo clic.

Los backups continuos (continuous backups) registran cada operacion de escritura en tiempo real, lo que permite realizar restauraciones point-in-time a cualquier momento dentro de la ventana de retencion. Esta modalidad utiliza el oplog de MongoDB para capturar los cambios y los almacena en un almacenamiento de objetos externo (S3, Azure Blob o GCS). Para activar los backups continuos es necesario un cluster dedicado (M10 o superior).

La restauracion point-in-time permite recuperar el cluster al estado exacto de un timestamp dado, con una granularidad de segundos. Es especialmente util para recuperarse de errores humanos (borrado accidental de datos, actualizaciones masivas incorrectas) sin perder las operaciones posteriores correctas.

> Los backups en Atlas no consumen almacenamiento del cluster, sino que se almacenan externamente. Esto implica que el espacio en disco del cluster se dedica exclusivamente a los datos activos.

### Monitorizacion

El panel de monitorizacion de Atlas ofrece metricas en tiempo real sobre el rendimiento del cluster: operaciones por segundo, latencia de lecturas y escrituras, uso de CPU, memoria, conexiones activas, tamano de los datos e hit rate de la cache de WiredTiger. Las metricas se muestran en graficos con granularidad de 10 segundos para la ultima hora, 5 minutos para el ultimo dia y 1 hora para periodos mas largos.

Atlas incluye un sistema de alertas configurable que notifica por correo electronico, Slack, PagerDuty o webhook cuando se superan umbrales predefinidos. Las alertas cubren CPU elevada, latencia alta, espacio en disco bajo, failovers, cambios en la configuracion del cluster y anomalias en las consultas.

El query profiler analiza las consultas lentas y muestra el plan de ejecucion, los indices utilizados, el numero de documentos examinados y el tiempo de ejecucion. Integrado con la vista de performance advisor, Atlas puede sugerir indices para optimizar las consultas detectadas como lentas.

### Seguridad

La seguridad en Atlas opera en multiples capas. La red se protege mediante IP whitelisting (solo direcciones IP autorizadas pueden conectar), VPC peering (conexion privada entre la VPC del cliente y el cluster Atlas) y private endpoints (conexion directa desde la VPC sin atravesar Internet publico). Para entornos que requieren aislamiento total, Atlas ofrece tambien el modo "no Internet" donde el cluster solo es accesible desde conexiones peered.

El cifrado en reposo utiliza AES-256 gestionado por el proveedor cloud o por el cliente mediante AWS KMS, Azure Key Vault o Google Cloud KMS. El cifrado en transito se activa mediante TLS 1.2+ en todas las conexiones, incluyendo la comunicacion interna entre nodos del replica set.

La autenticacion se realiza mediante SCRAM-SHA-256, certificados X.509, LDAP o autenticacion federada con proveedores como Okta, Azure AD o Google Workspace. Los roles y permisos se gestionan con el mismo sistema de roles de MongoDB, permitiendo definir privilegios por base de datos, coleccion o accion.

### Global clusters

Los global clusters permiten distribuir los datos en multiples regiones geograficas desde un mismo cluster. Utilizan la funcionalidad de sharding de MongoDB combinada con etiquetas de zona (zone tags) para que los datos de una region concreta se almacenen fisicamente en los nodos de esa region. Las lecturas se dirigen a la replica local mas cercana mediante la preferencia de lectura `nearest`, reduciendo la latencia para usuarios distribuidos globalmente.

Un global cluster se compone de un nodo primario y varios secundarios distribuidos en distintas regiones. Las escrituras se dirigen siempre a la region donde reside el primario, pero las lecturas pueden servirse desde cualquier replica. Para aplicaciones que requieren escrituras locales en multiples regiones, Atlas ofrece configuraciones multi-region con topologias personalizadas.

