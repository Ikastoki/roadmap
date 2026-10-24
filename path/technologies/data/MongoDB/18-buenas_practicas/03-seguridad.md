# Seguridad

La seguridad en MongoDB no es un atributo que se activa con un solo interruptor, sino una capa que se construye desde la autenticacion hasta el cifrado en reposo, pasando por el control de acceso y la auditoria. Descuidar cualquiera de estas capas expone los datos a accesos no autorizados, filtraciones o perdidas.

## Autenticacion habilitada

Por defecto, las instalaciones nuevas de MongoDB hasta la version 6.x arrancan sin autenticacion. Esto significa que cualquiera que alcance el puerto 27017 (o el configurado) tiene acceso completo a todos los datos. La primera medida de seguridad es habilitar la autenticacion y crear usuarios con credenciales fuertes.

```yaml
# mongod.conf
security:
   authorization: enabled
```

MongoDB soporta multiples mecanismos de autenticacion:

- **SCRAM** (Salted Challenge Response Authentication Mechanism): el metodo por defecto, basado en contrasena y sal. SCRAM-SHA-256 es la variante recomendada frente a SCRAM-SHA-1.
- **x.509**: autenticacion mediante certificados SSL/TLS, comun en entornos empresariales con PKI.
- **LDAP**: integracion con directorios corporativos (Active Directory, OpenLDAP) en MongoDB Enterprise.
- **Kerberos**: autenticacion delegada a un centro de distribucion de claves (KDC), tambien en Enterprise.

Para uso general, SCRAM con SHA-256 es suficiente. Entornos con politicas de acceso centralizadas se benefician de LDAP o Kerberos.

```javascript
use admin
db.createUser({
   user: "admin",
   pwd: passwordPrompt(),
   roles: [
      { role: "root", db: "admin" }
   ],
   mechanisms: ["SCRAM-SHA-256"]
})
```

> La contrasena se envia hasheada, no en texto plano. Nunca storear credenciales en codigo fuente; usar variables de entorno o secretos gestionados externamente (Vault, AWS Secrets Manager).

## Minimo privilegio

El principio de minimo privilegio establece que cada usuario debe tener exactamente los permisos que necesita para realizar su trabajo, ni uno mas. MongoDB implementa el control de acceso mediante roles que agrupan privilegios sobre recursos (bases de datos, colecciones, clusters).

Los roles predefinidos cubren los escenarios mas comunes:

- `read`: solo lectura en la base de datos especificada.
- `readWrite`: lectura y escritura.
- `dbAdmin`: administracion del esquema (indices, validacion).
- `userAdmin`: gestion de usuarios y roles.
- `clusterAdmin`: administracion del cluster (replicacion, sharding).
- `root`: superusuario con todos los privilegios.

Nunca usar `root` para operaciones rutinarias. Una aplicacion que solo necesita leer y escribir en `db_ventas` debe tener un usuario con rol `readWrite` sobre esa base de datos, no sobre `admin`.

```javascript
use db_ventas
db.createUser({
   user: "app_ventas",
   pwd: passwordPrompt(),
   roles: [
      { role: "readWrite", db: "db_ventas" }
   ]
})
```

Para escenarios mas finos, MongoDB permite crear roles personalizados con privilegios especificos: acceso a colecciones concretas, permisos para ejecutar determinados comandos o restriccion de operaciones por IP.

```javascript
use admin
db.createRole({
   role: "lector_facturas",
   privileges: [
      {
         resource: { db: "contabilidad", collection: "facturas" },
         actions: ["find"]
      }
   ],
   roles: []
})
```

## Cifrado en reposo y transito

El cifrado protege los datos cuando el atacante ha sorteado las capas de autenticacion y control de acceso. Opera en dos niveles:

- **Cifrado en transito (TLS/SSL)**: protege los datos mientras viajan entre el cliente y el servidor, y entre los miembros del replica set. Sin TLS, un atacante en la misma red puede interceptar el trafico con un sniffer y leer los datos en texto plano.

```yaml
# mongod.conf
net:
   tls:
      mode: requireTLS
      certificateKeyFile: /etc/mongodb/server.pem
      CAFile: /etc/mongodb/ca.pem
```

- **Cifrado en reposo (encryption at rest)**: protege los datos almacenados en disco. Si alguien obtiene los archivos de datos (por robo del disco, copia no autorizada o backup comprometido), no puede leerlos sin la clave de cifrado. MongoDB Enterprise incluye cifrado nativo en reposo con AES-256-GCM. En la edicion Community, puede implementarse mediante cifrado a nivel de sistema (LUKS en Linux, BitLocker en Windows).

```yaml
# mongod.conf (MongoDB Enterprise)
security:
   enableEncryption: true
   encryptionCipherMode: AES256-GCM
   encryptionKeyFile: /etc/mongodb/encryption-key
```

> La clave de cifrado debe gestionarse con un sistema externo (KMIP, Vault, AWS KMS) en lugar de almacenarse en el sistema de archivos local. Si la clave se pierde, los datos son irrecuperables.

## Auditoria

La auditoria registra cada operacion que afecta a la seguridad: inicios de sesion, cambios de permisos, creacion de usuarios, ejecucion de comandos administrativos. Es el unico mecanismo para detectar accesos no autorizados, errores de configuracion o actividad sospechosa tras un incidente.

```yaml
# mongod.conf
auditLog:
   destination: file
   format: JSON
   path: /var/log/mongodb/audit.log
   filter: '{ atype: { $in: ["authenticate", "createUser", "dropUser", "grantRolesToUser", "revokeRolesFromUser", "createRole", "dropRole", "dropDatabase", "dropCollection", "shutdown"] } }'
```

El filtro de auditoria permite seleccionar solo los eventos relevantes, evitando que el log crezca sin control. Los eventos tipicos a monitorizar son autenticaciones fallidas, cambios en roles y usuarios, y operaciones destructivas (drop, shutdown).

Los logs de auditoria deben protegerse con permisos de solo lectura para el usuario administrador y almacenarse en un sistema centralizado (SIEM, Elasticsearch, Splunk) donde puedan correlacionarse con otros eventos de seguridad.

## Rotacion de credenciales

Las contrasenas y claves tienen una vida util limitada. Cuanto mas tiempo permanece activa una credencial, mayor es la ventana para que sea comprometida sin ser detectada. La rotacion periodica reduce este riesgo.

Para los usuarios de base de datos, la rotacion implica actualizar la contrasena con `changeUserPassword`:

```javascript
use admin
db.changeUserPassword("app_ventas", passwordPrompt())
```

En entornos Kubernetes, la rotacion se integra con secretos: cada cierto tiempo se actualiza el secreto y la aplicacion recarga la configuracion sin reiniciar. Las claves de cifrado en reposo deben rotarse con el procedimiento de `rotateEncryptionKey` de MongoDB Enterprise.

> La rotacion no debe interrumpir el servicio. Las aplicaciones deben implementar reconexion automatica con las nuevas credenciales y tolerar breves periodos en los que el pool de conexiones mezcle credenciales antiguas y nuevas.

