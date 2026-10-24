# Seguridad de red y auditoria

La seguridad de una base de datos MongoDB no termina en la autenticacion y el cifrado. El control del acceso a nivel de red y la capacidad de registrar todas las operaciones relevantes constituyen capas defensivas esenciales para prevenir accesos no autorizados y realizar analisis forenses en caso de incidente.

## Control de acceso por red: bindIp

El parametro `net.bindIp` determina que interfaces de red escucha el proceso `mongod` para aceptar conexiones entrantes. Su valor por defecto ha evolucionado entre versiones: desde MongoDB 3.6, las distribuciones oficiales establecen `127.0.0.1` (localhost) como unico destino permitido, lo que restringe el acceso unicamente a conexiones locales.

```yaml
net:
  bindIp: "127.0.0.1"
```

Cuando se necesita acceso remoto, es necesario especificar las direcciones IP de las interfaces que deben aceptar conexiones:

```yaml
net:
  bindIp: "192.168.1.100,10.0.0.1"
```

El uso de `0.0.0.0` vincula el servidor a todas las interfaces de red disponibles, incluida la interfaz publica si existe. Esta configuracion expone MongoDB a internet y solo deberia emplearse cuando un firewall externo garantiza el filtrado del trafico.

> Nunca se debe exponer MongoDB directamente a internet sin un firewall o una VPN. El puerto 27017 es un objetivo habitual de ataques automatizados que buscan bases de datos sin proteger.

### Multiples interfaces y separacion de trafico

En despliegues con replica set, es habitual separar el trafico de cliente del trafico interno de replicacion. MongoDB permite especificar distintas interfaces segun el proposito:

```yaml
net:
  bindIp: "192.168.1.100"
  ipv6: false

replication:
  localPingThresholdMs: 15
```

Para entornos que requieren aislamiento adicional, se recomienda utilizar dos interfaces de red distintas: una para el trafico de aplicaciones y otra para la replicacion interna entre nodos del cluster.

## Firewalls y puerto 27017

MongoDB utiliza el puerto 27017 por defecto para las conexiones de cliente, mientras que el puerto 27018 se reserva para el trafico interno de sharding y el 27019 para los procesos `mongos` en entornos fragmentados.

Las reglas de firewall deben restringir el acceso al puerto 27017 exclusivamente a las direcciones IP de las aplicaciones que necesitan conectarse. En sistemas Linux con `iptables`:

```bash
iptables -A INPUT -p tcp --dport 27017 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -j DROP
```

Con `ufw` en Ubuntu:

```bash
ufw allow from 10.0.0.0/8 to any port 27017
ufw deny 27017
```

Para despliegues en la nube, los grupos de seguridad (AWS Security Groups, Azure NSG, GCP Firewall Rules) constituyen el mecanismo recomendado. Cada grupo de seguridad debe permitir el trafico entrante al puerto 27017 unicamente desde las subredes de las aplicaciones autorizadas.

> En replica sets, los nodos deben poder comunicarse entre si a traves del puerto 27017. Es necesario anadir reglas que permitan el trafico entre los miembros del conjunto, ademas de las reglas para clientes.

## Seguridad de red en MongoDB Atlas

MongoDB Atlas, el servicio de base de datos como servicio (DBaaS) de MongoDB, implementa varias capas de seguridad de red que abstraen la gestion manual de firewalls y direcciones IP.

### IP whitelisting

Atlas permite especificar una lista de direcciones IP o rangos CIDR autorizados para conectarse al cluster. Cualquier conexion desde una IP no incluida en la lista es rechazada a nivel de red, antes de llegar al proceso MongoDB.

La configuracion se realiza desde la consola web de Atlas o mediante la API:

```bash
curl -u "$PUBLIC_KEY:$PRIVATE_KEY" \
  --digest \
  --header "Content-Type: application/json" \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/$GROUP_ID/whitelist" \
  --data '{
    "whitelist": [
      {"ipAddress": "203.0.113.0/24", "comment": "Oficina central"}
    ]
  }'
```

### VPC Peering

Para entornos que requieren baja latencia y aislamiento total, Atlas soporta conexion directa con Virtual Private Clouds (VPC) en AWS, Azure y GCP mediante VPC peering. Esta conexion permite que los recursos en la nube del cliente se comuniquen con el cluster Atlas a traves de la red privada de la nube, sin atravesar internet.

El proceso de configuracion implica:

1. Crear un pending peering connection desde Atlas.
2. Aceptar la solicitud de peering desde la consola del proveedor de nube.
3. Configurar las tablas de enrutamiento para permitir el trafico entre ambas VPC.
4. Anadir los CIDR de la VPC del cliente a la whitelist de Atlas.

> El VPC peering reduce la latencia y elimina el coste de ancho de banda de salida hacia internet, pero requiere coordinacion entre el equipo de plataforma y el administrador de la nube.

### Private Endpoints

Atlas ofrece la posibilidad de conectar el cluster mediante AWS PrivateLink, Azure Private Link o Google Cloud Private Service Connect. Estos servicios crean una interfaz de red privada dentro de la VPC del cliente, eliminando la necesidad de que el trafico salga de la red privada.

## Auditoria

El sistema de auditoria de MongoDB registra operaciones administrativas y de acceso a datos, proporcionando un rastro verificable de todas las acciones ejecutadas sobre la base de datos. Esta funcionalidad solo esta disponible en MongoDB Enterprise.

### Activacion del audit logging

La auditoria se configura mediante el parametro `auditLog` en el archivo de configuracion:

```yaml
auditLog:
  destination: "file"
  format: "JSON"
  path: "/var/log/mongodb/audit.log"
  filter: '{ atype: { $in: ["authenticate", "createUser", "dropCollection", "dropDatabase"] } }'
```

El parametro `destination` acepta tres valores:

- **file**: escribe los eventos de auditoria a un archivo en el sistema de archivos local.
- **syslog**: envia los eventos al syslog del sistema operativo. Util cuando ya existe una infraestructura centralizada de logs.
- **console**: imprime los eventos en la salida estandar del proceso `mongod`. Adecuado para entornos de contenedores donde la salida se captura mediante el driver de logs del orquestador.

### Formato de salida

El formato `JSON` produce registros estructurados y facilmente procesables por herramientas como Logstash, Splunk o Elasticsearch. Cada entrada de auditoria incluye campos como `atype` (tipo de operacion), `ts` (marca de tiempo), `local` y `remote` (direcciones IP y puertos), `users` (usuarios involucrados) y `param` (parametros de la operacion).

Ejemplo de entrada de auditoria en formato JSON:

```json
{
  "atype": "authenticate",
  "ts": { "$date": "2026-07-28T10:30:00.000+0000" },
  "local": { "ip": "192.168.1.100", "port": 27017 },
  "remote": { "ip": "10.0.0.50", "port": 45000 },
  "users": [{ "user": "jperez", "db": "admin" }],
  "result": 0
}
```

El formato `BSON` genera un archivo binario que puede leerse posteriormente con la herramienta `mongoaudit`.

### Filtrado por operacion

El parametro `filter` permite seleccionar unicamente los eventos de interes, reduciendo el volumen de registros y el impacto en el rendimiento. El filtro utiliza la sintaxis de consulta de MongoDB y puede combinarse con operadores logicos:

```yaml
auditLog:
  destination: "file"
  format: "JSON"
  path: "/var/log/mongodb/audit.log"
  filter: '{
    $or: [
      { atype: { $in: ["createUser", "dropUser", "grantRolesToUser", "revokeRolesFromUser"] } },
      { atype: { $regex: "^drop" } }
    ]
  }'
```

Los tipos de evento (`atype`) mas relevantes incluyen:

| atype | Descripcion |
|---|---|
| `authenticate` | Intentos de autenticacion, exitosos o fallidos |
| `createUser` | Creacion de usuarios |
| `dropUser` | Eliminacion de usuarios |
| `grantRolesToUser` | Asignacion de roles |
| `createCollection` | Creacion de colecciones |
| `dropCollection` | Eliminacion de colecciones |
| `createIndex` | Creacion de indices |
| `dropIndex` | Eliminacion de indices |
| `killOp` | Terminacion forzada de operaciones |

> La auditoria tiene un impacto medible en el rendimiento, especialmente cuando se registran todas las operaciones sin filtro. Se recomienda monitorizar la latencia de escritura tras activarla y ajustar el filtro para capturar solo los eventos estrictamente necesarios.

