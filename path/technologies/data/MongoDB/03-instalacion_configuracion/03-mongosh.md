# mongosh

El shell de MongoDB ha evolucionado significativamente desde el antiguo `mongo` (basado en SpiderMonkey) hasta `mongosh`, un shell moderno construido sobre Node.js y Chromium V8. Esta reescritura no solo ofrece un interprete JavaScript mas potente, sino que incorpora autocompletado, resaltado de sintaxis, historial persistente y una API unificada que lo convierte en una herramienta tanto para administracion como para scripting complejo.

## Instalacion

mongosh se distribuye como un paquete independiente que puede instalarse junto a MongoDB o por separado.

```bash
# Ubuntu/Debian
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | \
    sudo apt-key add -
echo "deb http://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | \
    sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get update
sudo apt-get install -y mongodb-mongosh

# macOS
brew install mongosh

# Windows (PowerShell)
msiexec.exe /q /i mongosh-2.1.0-x64.msi
```

La instalacion independiente permite usar mongosh para conectarse a cualquier servidor MongoDB, incluso en entornos donde no se tiene control sobre la instalacion del servidor (como Atlas o despliegues gestionados).

## Conexion al servidor

La forma mas basica de conexion es sin argumentos, asumiendo `localhost:27017`:

```bash
mongosh
```

Para conexiones especificas:

```bash
mongosh "mongodb://localhost:27017/mi_bd"
mongosh "mongodb+srv://cluster.mongodb.net/mi_bd" --username admin
mongosh --host localhost --port 27017 --username admin --authenticationDatabase admin
```

> mongosh intenta autenticar automaticamente si se proporcionan credenciales. Si falla, presenta un shell sin autenticar; cualquier operacion devolvera un error de autorizacion.

Al conectarse, mongosh muestra informacion del servidor en el mensaje de bienvenida: version, nombre del replica set (si aplica) y la base de datos activa por defecto (`test`).

## Metacomandos

mongosh expone comandos que comienzan con `\` (backslash) para operaciones administrativas sin necesidad de usar JavaScript. Estos comandos son accesibles directamente desde el prompt.

| Comando      | Descripcion                                                |
|--------------|------------------------------------------------------------|
| `\help`      | Muestra la lista completa de metacomandos                  |
| `\use <db>`  | Cambia a la base de datos especificada                     |
| `\show dbs`  | Lista todas las bases de datos                             |
| `\show collections` | Lista las colecciones de la base de datos actual    |
| `\show users` | Muestra los usuarios de la base de datos actual           |
| `\show roles` | Muestra los roles de la base de datos actual              |
| `\show databases` | Alias de `\show dbs`                                   |
| `\db`        | Muestra la base de datos actual                            |
| `\exit`      | Cierra la conexion y sale del shell                        |
| `\history`   | Muestra el historial de comandos                           |
| `\load <file>` | Ejecuta un archivo JavaScript en el shell                |
| `\config`    | Abre el editor de configuracion del shell                  |

Los metacomandos son syntactic sugar sobre operaciones JavaScript. Por ejemplo, `\show dbs` equivale a `db.adminCommand('listDatabases')`.

## Autocompletado y ayuda contextual

mongosh incorpora autocompletado interactivo mediante la tecla Tab. El shell sugiere:

- Nombres de bases de datos y colecciones
- Metodos de los objetos `db`, `db.<coleccion>` y `DBCollection`
- Propiedades de objetos JavaScript
- Nombres de metacomandos

Ademas, el metodo `help()` proporciona documentacion contextual:

```javascript
db.help()               // Ayuda sobre el objeto db
db.mi_coleccion.help()  // Ayuda sobre metodos de coleccion
db.mi_coleccion.find().help() // Ayuda sobre el cursor
```

## Scripting con JavaScript

mongosh ejecuta JavaScript V8 con soporte completo de ES2020+. Esto permite escribir scripts complejos combinando operaciones de base de datos con logica de programacion tradicional.

```javascript
// Script: migracion.js
const db = connect("mongodb://localhost:27017/mi_bd");

const usuarios = db.usuarios.find({ activo: true }).toArray();

for (const usuario of usuarios) {
  const total = db.pedidos.countDocuments({ usuario_id: usuario._id });
  db.usuarios.updateOne(
    { _id: usuario._id },
    { $set: { total_pedidos: total } }
  );
  print(`Actualizado ${usuario.nombre}: ${total} pedidos`);
}
```

Ejecucion desde la terminal:

```bash
mongosh migracion.js
```

O desde el shell:

```javascript
\load migracion.js
```

### try/catch y manejo de errores

mongosh permite manejo estructurado de errores:

```javascript
try {
  db.mi_coleccion.insertOne({ _id: 1, valor: "test" });
} catch (e) {
  console.error("Error al insertar:", e.message);
}
```

### Uso de API modernas

Al estar basado en Node.js, mongosh soporta APIs como `console.log`, `console.table`, `JSON.stringify` con formato, y desestructuracion:

```javascript
const { name, email } = db.usuarios.findOne({ _id: ObjectId("...") });
print(`Usuario: ${name} <${email}>`);

console.table(db.usuarios.find().limit(5).toArray());
```

## Configuracion del shell

El comando `\config` abre un menu interactivo para personalizar el comportamiento del shell:

- **editor**: editor de texto externo para escribir comandos multilinea
- **banner**: mostrar/ocultar el mensaje de bienvenida
- **prompt**: personalizar el prompt del shell
- **historyLength**: numero de comandos en el historial

Los cambios se persisten en `~/.mongodb/mongoshrc.json`:

```json
{
  "editor": "vim",
  "banner": false,
  "historyLength": 500
}
```

## Modo REPL silencioso

mongosh puede ejecutarse en modo silencioso para scripting automatizado, suprimiendo el banner y los mensajes informativos:

```bash
mongosh --quiet --eval "db.version()"
```

La opcion `--eval` ejecuta una expresion JavaScript directamente desde la linea de comandos y finaliza el shell. Combinado con `--quiet`, es util para pipelines de automatizacion y monitoreo.

