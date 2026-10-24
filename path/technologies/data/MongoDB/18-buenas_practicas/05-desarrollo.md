# Desarrollo

Las decisiones que se toman en la capa de aplicacion determinan en buena medida la salud de MongoDB en produccion. Un ODM bien configurado, una validacion rigurosa y un manejo correcto de errores y paginacion evitan problemas que, una vez en produccion, son dificiles de corregir sin migraciones o downtime.

## ODMs (Object-Document Mappers)

Los ODMs como Mongoose (Node.js), Spring Data MongoDB (Java) o Morphia traducen entre el modelo de objetos de la aplicacion y los documentos BSON de MongoDB. Su principal ventaja es la productividad: permiten definir esquemas, validaciones y relaciones con codigo declarativo.

Mongoose, por ejemplo, define un esquema que se mapea a una coleccion:

```javascript
const usuarioSchema = new mongoose.Schema({
   nombre:    { type: String, required: true, trim: true },
   email:     { type: String, required: true, unique: true, lowercase: true },
   edad:      { type: Number, min: 0, max: 150 },
   roles:     { type: [String], enum: ["admin", "user", "moderator"] },
   fecha_reg: { type: Date, default: Date.now }
}, { timestamps: true })

module.exports = mongoose.model("Usuario", usuarioSchema)
```

```java
// Spring Data MongoDB
@Document(collection = "usuarios")
public class Usuario {
   @Id private String id;
   @NotNull @Indexed(unique = true) private String email;
   @Min(0) @Max(150) private Integer edad;
   private List<String> roles;
   @CreatedDate private Instant fechaReg;
}
```

El ODM abstrae la base de datos, pero introduce riesgos si no se usa con cuidado. La principal trampa es el "N+1 problem": al acceder a una propiedad referenciada, el ODM lanza una consulta adicional por cada elemento, multiplicando las operaciones de red. En Mongoose, el uso de `.populate()` sin indices adecuados o sin planificar las referencias que se necesitan puede degradar el rendimiento.

La regla es: usar el ODM para definir esquemas y validaciones en desarrollo, pero conocer las operaciones nativas de MongoDB para las rutas criticas y monitorear las queries que genera.

## Validacion de esquemas en la aplicacion

La validacion con `$jsonSchema` en el servidor es la ultima linea de defensa, pero la primera debe estar en la aplicacion. Validar los datos antes de enviarlos a la base de datos evita escrituras fallidas, reduce la carga del servidor y proporciona mensajes de error mas utiles al usuario.

Las librerias de validacion (Joi, Yup, Zod en Node.js; Bean Validation en Java) permiten definir reglas complejas que `$jsonSchema` no soporta: comprobaciones entre campos, formato de documentos segun el estado de la entidad o reglas de negocio que dependen de datos externos.

```javascript
// Validacion en aplicacion con Zod
const usuarioSchema = z.object({
   nombre: z.string().min(2).max(100),
   email: z.string().email(),
   edad: z.number().int().min(0).max(150).optional(),
   tipo: z.enum(["admin", "user"])
})

function crearUsuario(data) {
   const parsed = usuarioSchema.parse(data) // lanza error si no valida
   return db.collection("usuarios").insertOne(parsed)
}
```

La doble validacion (aplicacion + base de datos) protege contra clientes maliciosos, errores en la logica de la aplicacion y accesos administrativos directos que bypassean la capa de validacion.

## Manejo de errores por duplicados

Cuando un campo tiene un indice unico y la aplicacion intenta insertar un valor duplicado, MongoDB lanza un error con codigo `11000` (DuplicateKey). Este error debe capturarse explicitamente y traducirse a un mensaje significativo para el usuario, no mostrarse como un error interno del servidor.

```javascript
try {
   await db.collection("usuarios").insertOne({ email: "existente@mail.com" })
} catch (err) {
   if (err.code === 11000) {
      // El campo repetido esta en err.keyPattern
      const campo = Object.keys(err.keyPattern)[0]
      throw new Error(`El ${campo} ya esta registrado`)
   }
   throw err // otros errores se relanzan
}
```

Un error comun es asumir que la comprobacion previa de existencia (`findOne` antes de insertar) evita el duplicado. En entornos concurrentes, dos procesos pueden ejecutar `findOne` simultaneamente, ambos obtener `null` e insertar el mismo valor. Solo el indice unico y la captura del error `11000` garantizan la integridad.

## Transacciones solo cuando son necesarias

Desde MongoDB 4.0, las transacciones multi-documento con garantias ACID estan disponibles. Sin embargo, usarlas indiscriminadamente es contraproducente: las transacciones distribuidas tienen un coste en latencia y rendimiento significativamente mayor que las operaciones atomicas individuales.

La mayoria de las operaciones que requieren atomicidad pueden resolverse con operaciones atomicas de un solo documento. MongoDB garantiza que las operaciones sobre un unico documento (insertOne, updateOne, findAndModify) son atomicas: o se aplican completamente o no se aplican.

```javascript
// Atomico y sin transaccion: decrementar stock si hay suficiente
const result = await db.collection("productos").findOneAndUpdate(
   { _id: productoId, stock: { $gte: cantidad } },
   { $inc: { stock: -cantidad } },
   { returnDocument: "after" }
)
if (!result) {
   throw new Error("Stock insuficiente")
}
```

Solo cuando la operacion abarca varios documentos y requiere que todos se actualicen o ninguno (por ejemplo, transferencia entre cuentas con debito y credito en documentos separados) tiene sentido usar una transaccion.

```javascript
const session = client.startSession()
try {
   session.startTransaction()
   await db.collection("cuentas").updateOne(
      { _id: origen }, { $inc: { saldo: -monto } }, { session }
   )
   await db.collection("cuentas").updateOne(
      { _id: destino }, { $inc: { saldo: +monto } }, { session }
   )
   await session.commitTransaction()
} catch (err) {
   await session.abortTransaction()
   throw err
} finally {
   session.endSession()
}
```

> Las transacciones en MongoDB no son gratuitas. Incrementan la latencia, requieren coordinacion entre nodos y pueden fallar por conflictos de escritura. Usarlas solo cuando la logica de negocio lo exige.

## Paginacion correcta

La paginacion basada en `skip()` y `limit()` es la mas sencilla de implementar, pero tambien la mas ineficiente a medida que aumenta el numero de paginas. Cada llamada con `skip(10000).limit(20)` obliga a MongoDB a recorrer y descartar los primeros 10000 documentos, un coste que crece linealmente con la profundidad de la pagina.

```javascript
// Ineficiente en paginas profundas
db.productos.find({ categoria: "electronica" })
   .sort({ precio: -1 })
   .skip(10000)
   .limit(20)
```

La alternativa correcta es la paginacion cursor-based (tambien llamada seek method). En lugar de contar documentos saltados, se utiliza el ultimo valor de la pagina anterior como punto de partida:

```javascript
// Eficiente: solo devuelve los 20 siguientes
db.productos.find({
   categoria: "electronica",
   precio: { $lt: ultimo_precio }  // ultimo_precio de la pagina anterior
}).sort({ precio: -1 }).limit(20)
```

Para ordenaciones por varios campos, el cursor debe combinar todos los campos de ordenacion:

```javascript
// Pagina anterior termino en { precio: 100, _id: ObjectId("...") }
db.productos.find({
   categoria: "electronica",
   $or: [
      { precio: { $lt: 100 } },
      { precio: 100, _id: { $gt: ObjectId("...") } }
   ]
}).sort({ precio: -1, _id: 1 }).limit(20)
```

La paginacion cursor-based tiene dos requisitos: el campo de ordenacion debe estar indexado y la aplicacion debe mantener el estado del cursor (tipicamente en el cuerpo de la respuesta de la API REST). A cambio, el coste por pagina es constante independientemente de la profundidad, y los resultados son consistentes aunque se inserten nuevos documentos durante la paginacion.

> Evitar la paginacion basada en `skip` para conjuntos de datos que superen las primeras decenas de paginas. Para aplicaciones con miles de paginas, la paginacion cursor-based no es solo una optimizacion: es la unica opcion viable.

