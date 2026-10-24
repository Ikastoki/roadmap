# Autorizacion y roles en MongoDB

Una vez autenticado, cada usuario queda sujeto a un conjunto de privilegios determinados por los roles que se le han asignado. MongoDB implementa un modelo de autorizacion basado en RBAC (Role-Based Access Control), donde los permisos se agrupan en roles y estos se asignan a usuarios o a otros roles mediante herencia. La autorizacion se evalua en cada operacion que el usuario intenta ejecutar contra la base de datos.

## Roles built-in

MongoDB proporciona un conjunto de roles predefinidos que cubren los patrones de acceso mas comunes. Estos roles se clasifican por ambito de aplicacion: roles a nivel de base de datos y roles a nivel de cluster.

### Roles de base de datos

Cada base de datos, incluyendo `admin`, dispone de los siguientes roles nativos:

- **read**: concede permiso de lectura sobre todas las colecciones de la base de datos. Incluye operaciones como `find`, `aggregate` y `listCollections`.
- **readWrite**: hereda todos los permisos de `read` y anade capacidad de escritura (`insert`, `update`, `delete`, `createCollection`, `dropCollection`).
- **dbAdmin**: permite realizar tareas administrativas sobre la base de datos sin conceder permisos de lectura o escritura sobre los datos. Incluye `collMod`, `createIndex`, `dropDatabase` y la ejecucion de comandos como `validate` o `compact`.
- **userAdmin**: permite crear y modificar usuarios y roles dentro de la base de datos. No concede permisos sobre los datos, pero si la capacidad de otorgar permisos a otros usuarios.
- **dbOwner**: combina los privilegios de `readWrite`, `dbAdmin` y `userAdmin` sobre la base de datos. Es el rol mas permisivo a nivel de base de datos.

### Roles de cluster

Los roles de cluster se definen en la base de datos `admin` y afectan a operaciones que involucran a todo el sistema:

- **clusterAdmin**: concede el maximo nivel de acceso administrativo sobre el cluster. Combina los privilegios de `clusterManager`, `clusterMonitor` y `hostManager`, permitiendo operaciones como `addShard`, `replSetReconfig` o `enableSharding`.
- **clusterManager**: permite administrar operaciones de replica set y sharding, incluyendo `replSetInitiate`, `replSetReconfig` y `resync`.
- **clusterMonitor**: acceso de solo lectura a informacion de monitoreo del cluster, como `serverStatus`, `currentOp` o `dbStats`.
- **hostManager**: permite operaciones de gestion del servidor como `killOp`, `shutdown` o `logRotate`.
- **root**: superusuario que acumula todos los privilegios de todos los roles existentes. Incluye `readWriteAnyDatabase`, `dbAdminAnyDatabase`, `userAdminAnyDatabase`, `clusterAdmin` y `restore`.

La asignacion de un rol se realiza durante la creacion del usuario o mediante `db.grantRolesToUser()`:

```javascript
use admin
db.createUser({
  user: "admin_plataforma",
  pwd: passwordPrompt(),
  roles: [
    { role: "root", db: "admin" },
    { role: "readWrite", db: "logs" }
  ]
})
```

> El rol `root` debe utilizarse con extrema precaucion. En entornos productivos es preferible asignar roles especificos a cada usuario o aplicacion siguiendo el principio de minimo privilegio.

## Roles personalizados

Cuando los roles built-in no se ajustan exactamente al modelo de permisos requerido, MongoDB permite crear roles personalizados que agrupan acciones especificas sobre recursos determinados.

Un rol personalizado se define mediante `db.createRole()`, especificando los privilegios y, opcionalmente, los roles de los que hereda:

```javascript
use ventas
db.createRole({
  role: "visor_informes",
  privileges: [
    {
      resource: { db: "ventas", collection: "informes" },
      actions: ["find", "aggregate"]
    },
    {
      resource: { db: "ventas", collection: "facturas" },
      actions: ["find"]
    }
  ],
  roles: [
    { role: "read", db: "ventas" }
  ]
})
```

Las acciones disponibles se organizan en categorias como acciones de escritura (`insert`, `update`, `remove`), acciones administrativas (`createCollection`, `dropIndex`, `enableSharding`) o acciones de monitoreo (`serverStatus`, `top`). La documentacion oficial de MongoDB mantiene la lista completa de acciones reconocidas.

Para asignar un rol personalizado a un usuario se utiliza el mismo mecanismo que con los roles built-in:

```javascript
db.grantRolesToUser("jperez", [{ role: "visor_informes", db: "ventas" }])
```

Los roles personalizados se almacenan en la coleccion `system.roles` de la base de datos donde fueron creados. Al eliminar una base de datos se eliminan tambien todos los roles personalizados definidos en ella.

> Los nombres de los roles personalizados no deben comenzar con el prefijo `db_` o `cluster_`, que estan reservados para roles internos de MongoDB.

## Base de datos de usuarios administradores

La base de datos `admin` ocupa una posicion especial en el modelo de autorizacion de MongoDB. Los usuarios creados en `admin` pueden tener privilegios que abarquen a todo el cluster, mientras que los usuarios definidos en otras bases de datos solo pueden tener permisos limitados a esa base de datos.

Cualquier usuario con el rol `userAdminAnyDatabase` en `admin` puede gestionar usuarios en cualquier base de datos del sistema:

```javascript
use admin
db.createUser({
  user: "gestor_usuarios",
  pwd: passwordPrompt(),
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" }
  ]
})
```

La autenticacion siempre especifica la base de datos `admin` como `authenticationDatabase` cuando el usuario fue creado en ella. Si el usuario se creo en otra base de datos, esa base de datos actua como `authenticationDatabase`:

```bash
mongosh -u "jperez" -p --authenticationDatabase "ventas"
```

> El flag `--authenticationDatabase` es obligatorio cuando el usuario no fue creado en `admin`. Omite especificarlo cuando el usuario pertenece a `admin`, ya que MongoDB lo utiliza por defecto.

