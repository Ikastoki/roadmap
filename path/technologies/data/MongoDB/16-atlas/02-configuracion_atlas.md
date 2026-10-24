# Configuracion de Atlas

La puesta en marcha de un cluster en Atlas implica varios pasos secuenciales que van desde la creacion del entorno hasta la obtencion de la cadena de conexion que las aplicaciones utilizaran para interactuar con la base de datos. Aunque la interfaz web de Atlas guia al usuario a traves del proceso, comprender cada componente permite tomar decisiones informadas sobre seguridad, rendimiento y costes.

## Creacion del cluster

El punto de entrada a Atlas es la consola web en `cloud.mongodb.com`. Tras crear una cuenta e iniciar sesion, el asistente de creacion de clusters solicita tres decisiones fundamentales: el proveedor cloud, la region y el tamano del cluster.

La eleccion del proveedor cloud (AWS, Azure o GCP) depende de la infraestructura existente en la organizacion. Si la aplicacion ya se ejecuta en AWS, tiene sentido elegir AWS para evitar costes de transferencia de datos entre nubes y minimizar la latencia. Atlas ofrece las mismas funcionalidades en los tres proveedores, aunque algunas capacidades avanzadas como global clusters pueden tener disponibilidad regional distinta.

La region debe seleccionarse lo mas cerca posible de los usuarios o servidores de la aplicacion. Atlas muestra la latencia relativa de cada region durante la seleccion. Para aplicaciones con usuarios distribuidos, los global clusters o nodos de lectura en regiones adicionales resuelven la latencia de lectura, pero las escrituras siguen yendo al primario.

El tamano del cluster determina la capacidad de procesamiento y almacenamiento. Para entornos de desarrollo, el tier gratuito M0 es suficiente. Para produccion, la eleccion entre M10, M20 o superiores depende de la carga de trabajo esperada y del presupuesto. Atlas permite cambiar el tamano en cualquier momento, aunque las operaciones de escalado pueden implicar un breve periodo de indisponibilidad en clusters sin alta disponibilidad configurada.

> En el asistente de creacion, la opcion "Additional Settings" permite configurar la version de MongoDB, el proveedor de backups, el cifrado en reposo y las etiquetas de facturacion. Una vez creado el cluster, estos parametros no pueden modificarse en el tier gratuito.

## Database users

Atlas no utiliza los usuarios del sistema operativo ni los usuarios de MongoDB locales de la instalacion estandar. En lugar de ello, gestiona sus propios usuarios de base de datos a traves de la consola web o la API. Cada usuario se asocia a un mecanismo de autenticacion (SCRAM-SHA-256 por defecto) y a uno o varios roles que determinan los permisos.

Los roles disponibles son los mismos que en MongoDB estandar: `read`, `readWrite`, `dbAdmin`, `dbOwner`, `clusterAdmin` y `root`, ademas de roles personalizados. La practica recomendada es crear usuarios con el minimo privilegio necesario. Para una aplicacion tipica, un usuario con rol `readWrite` sobre la base de datos de la aplicacion es suficiente; el usuario administrativo con rol `root` debe usarse solo para tareas de gestion.

La creacion de un usuario se realiza desde la pestana "Database Access" en la consola de Atlas. Se introduce el nombre de usuario, la contrasena (o certificado X.509) y se seleccionan los roles. Atlas almacena las credenciales cifradas y las expone unicamente en el momento de la creacion; si se pierde la contrasena, debe generarse una nueva.

## IP whitelist

Por defecto, Atlas bloquea todas las conexiones entrantes al cluster. Para permitir el acceso, es necesario anadir las direcciones IP o rangos CIDR desde los que se conectara la aplicacion. Este mecanismo se denomina IP whitelist o lista de acceso.

La configuracion se realiza en la pestana "Network Access". Cada entrada puede ser una IP individual (`203.0.113.5`), un rango CIDR (`203.0.113.0/24`) o la direccion IP actual del navegador (util para pruebas rapidas). Para entornos de desarrollo donde la IP cambia frecuentemente, se recomienda usar un rango CIDR amplio o un VPN con IP fija.

Una entrada especial es `0.0.0.0/0`, que permite conexiones desde cualquier direccion IP. Aunque Atlas recomienda no usarla en produccion, es practica comun en entornos serverless o cuando las aplicaciones se ejecutan desde IPs impredecibles. En ese caso, la seguridad recae enteramente en la autenticacion del usuario y el cifrado TLS.

> Las reglas de IP whitelist se aplican a nivel de proyecto de Atlas, no de cluster. Esto significa que una misma lista de acceso sirve para todos los clusters del proyecto, lo que simplifica la gestion en entornos con multiples entornos (desarrollo, staging, produccion).

## Connection strings

Una vez creado el cluster, los usuarios configurados y la whitelist de IP establecida, Atlas genera una cadena de conexion que los drivers de MongoDB utilizan para conectar. El formato para Atlas es siempre `mongodb+srv://`, que utiliza DNS SRV para resolver automaticamente los nodos del replica set.

```
mongodb+srv://<usuario>:<contrasena>@<cluster>.mongodb.net/<base_datos>?retryWrites=true&w=majority
```

El segmento `mongodb+srv://` indica al driver que debe resolver el nombre DNS mediante registros SRV para obtener la lista completa de nodos del replica set. Esto evita tener que especificar manualmente cada host y puerto. Si el cluster se escala o cambia la topologia, los registros SRV se actualizan y los drivers se adaptan sin modificar la cadena de conexion.

Los parametros de query que acompanan la URL son importantes para el comportamiento del driver:

- **retryWrites=true**: permite que el driver reintente automaticamente las escrituras si el primario falla durante la operacion. Sin este parametro, un failover durante una escritura provocaria un error en la aplicacion.
- **w=majority**: exige que la escritura sea confirmada por la mayoria de los nodos del replica set antes de devolver el exito. Proporciona la maxima consistencia, aunque incrementa la latencia.
- **maxPoolSize**: controla el numero maximo de conexiones simultaneas que el driver mantiene en el pool. El valor optimo depende de la concurrencia de la aplicacion.

Cada driver tiene su propia forma de configurar estos parametros. En Node.js se pasan como opciones del objeto `MongoClient`, en Java se configuran en `MongoClientSettings`, y en Python se anaden al diccionario de parametros de `MongoClient`.

```javascript
const { MongoClient } = require('mongodb');
const uri = 'mongodb+srv://usuario:password@cluster.mongodb.net/mi_db?retryWrites=true&w=majority';
const client = new MongoClient(uri);
await client.connect();
```

```python
from pymongo import MongoClient
uri = 'mongodb+srv://usuario:password@cluster.mongodb.net/mi_db?retryWrites=true&w=majority'
client = MongoClient(uri)
db = client.mi_db
```

> Las contrasenas en la cadena de conexion deben codificarse con percent-encoding si contienen caracteres especiales como `@`, `:` o `/`. Por ejemplo, una contrasena `P@ssw:rd` se representa como `P%40ssw%3Ard`.

## Data API

La Data API de Atlas expone una interfaz HTTPS RESTful que permite realizar operaciones CRUD sobre los datos sin necesidad de utilizar el protocolo wire de MongoDB ni instalar un driver. Esta API esta disenada para entornos serverless, funciones cloud, integraciones con herramientas de automatizacion y cualquier escenario donde no sea practico mantener una conexion persistente con el driver nativo.

La Data API se activa desde la pestana "Data API" en la consola de Atlas. Una vez habilitada, se genera una clave de API que debe incluirse en el header `apiKey` de cada peticion. Las peticiones se realizan a una URL con el formato:

```
https://data.mongodb-api.com/app/<app_id>/endpoint/data/v1/action/<accion>
```

Las acciones disponibles incluyen `find`, `findOne`, `insertOne`, `insertMany`, `updateOne`, `updateMany`, `replaceOne`, `deleteOne`, `deleteMany` y `aggregate`. Cada peticion es un POST con un cuerpo JSON que especifica la base de datos, la coleccion y los parametros de la operacion.

```bash
curl --request POST \
  --url 'https://data.mongodb-api.com/app/<app_id>/endpoint/data/v1/action/find' \
  --header 'apiKey: <api_key>' \
  --header 'Content-Type: application/json' \
  --data '{
    "dataSource": "Cluster0",
    "database": "mi_bd",
    "collection": "usuarios",
    "filter": { "activo": true },
    "limit": 10
  }'
```

La Data API es particularmente util en combinacion con AWS Lambda, Cloudflare Workers o Vercel Edge Functions, donde las funciones serverless no mantienen estado y establecer una conexion MongoDB nativa en cada invocacion resultaria ineficiente. Sin embargo, la API tiene limitaciones: no soporta change streams, no permite operaciones administrativas y la latencia es superior a la de una conexion directa debido a la sobrecarga HTTP.

Para aplicaciones que requieren la maxima eficiencia, la conexion nativa con el driver de MongoDB sigue siendo la opcion recomendada. La Data API funciona mejor como complemento para casos de uso puntuales donde la simplicidad de integracion prima sobre el rendimiento.

