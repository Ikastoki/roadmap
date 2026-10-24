# Configuracion de MongoDB

El comportamiento de `mongod` se define mediante un archivo de configuracion en formato YAML. A diferencia de otros sistemas que utilizan argumentos de linea de comandos como unica via, MongoDB centraliza toda la configuracion en `mongod.conf`, separando parametros de red, almacenamiento, registro y seguridad en secciones independientes. Comprender cada directiva permite ajustar el servidor al hardware y los requisitos de cada despliegue.

## Estructura de mongod.conf

El archivo de configuracion se localiza en `/etc/mongod.conf` en sistemas Linux y macOS, o en `C:\Program Files\MongoDB\Server\7.0\bin\mongod.cfg` en Windows. Su estructura YAML organiza las opciones en bloques jerarquicos:

```yaml
# mongod.conf - Configuracion basica
storage:
  dbPath: "/var/lib/mongodb"

systemLog:
  destination: file
  path: "/var/log/mongodb/mongod.log"
  logAppend: true

net:
  port: 27017
  bindIp: "127.0.0.1"

security:
  authorization: "disabled"

processManagement:
  timeZoneInfo: "/usr/share/zoneinfo"
```

Cada seccion agrupa directivas funcionalmente relacionadas. MongoDB lee este archivo al arrancar; si alguna directiva es incorrecta, el servidor rechaza iniciarse e indica el error en el log.

### net.port

El puerto por defecto es el **27017**, puerto registrado oficialmente para MongoDB en IANA. Aunque se puede cambiar, las herramientas cliente (mongosh, controladores) lo usan como valor predeterminado, por lo que cambiarlo obliga a especificarlo en cada conexion.

```yaml
net:
  port: 27018
```

### net.bindIp

Controla que interfaces de red escucha el servidor. El valor `127.0.0.1` restringe las conexiones a la maquina local. Para permitir conexiones remotas desde una IP especifica:

```yaml
net:
  bindIp: "192.168.1.100"
```

Para escuchar en todas las interfaces (no recomendado en produccion sin autenticacion):

```yaml
net:
  bindIp: "0.0.0.0"
```

> Vincular a `0.0.0.0` sin habilitar autenticacion expone la base de datos a toda la red. En entornos de produccion es preferible usar `bindIp` especifica mas un firewall restrictivo.

### storage.dbPath

Define el directorio donde MongoDB almacena los archivos de datos. La eleccion de este directorio impacta directamente en el rendimiento debido a la latencia del sistema de archivos subyacente.

```yaml
storage:
  dbPath: "/var/lib/mongodb"
  journal:
    enabled: true
```

La opcion `journal.enabled: true` activa el journaling, mecanismo que garantiza la durabilidad de las escrituras incluso ante un fallo del servidor. El journal se almacena dentro del mismo `dbPath` en un subdirectorio `journal/`.

### systemLog.path

Define la ubicacion del archivo de registro. El sistema de logs de MongoDB es fundamental para diagnosticar problemas de rendimiento, errores de conexion y fallos internos.

```yaml
systemLog:
  destination: file
  path: "/var/log/mongodb/mongod.log"
  logAppend: true
  logRotate: "reopen"
```

`logAppend: true` anade las entradas nuevas al final del archivo existente en lugar de sobrescribirlo. `logRotate: "reopen"` permite la rotacion de logs externa mediante `SIGUSR1` sin reiniciar el servidor.

## Autenticacion

Por defecto, MongoDB arranca sin autenticacion (`security.authorization: "disabled"`). Cualquier cliente que alcance el puerto 27017 tiene acceso completo a todas las bases de datos. La autenticacion se habilita tras crear al menos un usuario administrador.

### Creacion del usuario administrador

Conectado a un servidor sin autenticacion, se crea el primer usuario en la base de datos `admin`:

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [ "root" ]
})
```

El rol `root` otorga todos los privilegios sobre todas las bases de datos. `passwordPrompt()` solicita la contrasena de forma interactiva, evitando que quede registrada en el historial del shell.

### Activacion de la autenticacion

Una vez creado el usuario, se edita `mongod.conf`:

```yaml
security:
  authorization: "enabled"
```

Y se reinicia el servicio:

```bash
sudo systemctl restart mongod
```

A partir de este momento, toda conexion requiere credenciales validas:

```bash
mongosh -u admin -p --authenticationDatabase admin
```

El parametro `--authenticationDatabase` indica en que base de datos esta registrado el usuario. Sin este parametro, mongosh intenta autenticar contra la base de datos por defecto (`test`).

### Mecanismo de autenticacion: SCRAM-SHA-256

MongoDB utiliza **SCRAM-SHA-256** (Salted Challenge Response Authentication Mechanism) como mecanismo de autenticacion por defecto desde la version 4.0. SCRAM es un protocolo criptografico que prueba el conocimiento de la contrasena sin transmitirla por la red.

El proceso funciona en cuatro pasos:

1. El cliente solicita iniciar sesion enviando su nombre de usuario.
2. El servidor responde con un `salt` (valor aleatorio) y un `iteration count` (por defecto 15000).
3. El cliente calcula un `proof` usando PBKDF2 con el salt y las iteraciones, y lo envia al servidor.
4. El servidor verifica el proof contra el hash almacenado y responde con su propio proof para que el cliente tambien verifique la identidad del servidor (verificacion mutua).

Este mecanismo evita ataques de diccionario offline porque el servidor nunca expone el hash directamente, y previene ataques de repeticion porque cada intento de autenticacion utiliza un salt diferente.

> Aunque SCRAM-SHA-256 es el mecanismo por defecto, MongoDB mantiene compatibilidad con SCRAM-SHA-1 para clientes antiguos. La configuracion `authenticationMechanisms: "SCRAM-SHA-256,SCRAM-SHA-1"` en mongod.conf permite ambos.

## Verificacion del estado del servidor

Una vez configurado, el comando `db.serverStatus()` en mongosh proporciona informacion detallada sobre el estado del servidor, incluyendo la version, el uptime, las conexiones activas y el estado de la autenticacion.

```javascript
db.serverStatus().security
// {
//   "SSLServerSubjectName": "",
//   "SSLServerHasCertificateAuthority": false,
//   "authenticationMechanisms": ["SCRAM-SHA-256", "SCRAM-SHA-1"]
// }
```

Para verificar que la autenticacion esta activa:

```javascript
db.runCommand({ connectionStatus: 1 })
```

El campo `authInfo.authenticatedUsers` lista los usuarios autenticados en la conexion actual. Si esta vacio y `security.authorization` esta en `enabled`, las operaciones fallaran con error de autorizacion.

