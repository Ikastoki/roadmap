# Diseno de esquemas

El diseno de esquemas en MongoDB determina en mayor medida que en SQL el rendimiento y la mantenibilidad de una aplicacion. En un sistema relacional, el esquema se normaliza primero y las consultas se adaptan a el; en MongoDB, el esquema se disena a partir de las consultas que la aplicacion necesita responder. Este cambio de mentalidad -esquema al servicio de las queries, no al reves- constituye la primera y mas importante buena practica.

## Modelar para queries

Antes de escribir un solo documento, hay que identificar los patrones de acceso de la aplicacion: que datos se leen juntos, con que frecuencia se actualizan, cuales son los queries mas comunes y que latencia es aceptable. Este analisis determina si los datos deben ir embebidos en el mismo documento o referenciados en colecciones separadas.

La regla general es embeder datos cuando la relacion es de uno-a-pocos y los datos se leen siempre juntos. Por ejemplo, las direcciones de envio de un pedido rara vez tienen sentido sin el pedido mismo, por lo que es natural incluirlas como un array embebido. Por el contrario, las relaciones uno-a-muchos o muchos-a-muchos donde los datos se consultan independientemente (como usuarios y sus comentarios) se benefician de la referencia mediante ObjectId.

El embedding reduce la necesidad de joins (que en MongoDB se implementan con `$lookup`, una operacion costosa) y garantiza que los datos relacionados se lean en una unica operacion de disco. Sin embargo, el embedding excesivo produce documentos demasiado grandes que se actualizan con frecuencia, lo que obliga al motor a reescribir el documento completo aunque solo haya cambiado un campo del array.

## Limite de 16 MB

Cada documento en MongoDB tiene un limite de tamano de 16 MB. Este limite no es arbitrario: responde a la necesidad de mantener el rendimiento de la red y la memoria, evitando que una unica operacion de lectura o escritura monopolice los recursos. Un documento que se acerca a este limite es sintoma de un diseno inadecuado.

Los arrays que crecen sin control (arrays infinitos) son la causa principal de documentos que superan los 16 MB. Un array de mensajes en un chat, de transacciones en una cuenta bancaria o de lecturas de un sensor crece indefinidamente con el tiempo. En estos casos, la solucion es promocionar el array a una coleccion separada donde cada elemento sea un documento independiente referenciado por un campo `parent_id`.

> MongoDB no ofrece un warning cuando un documento se acerca a los 16 MB; la escritura falla directamente con un error `BSONObjectTooLarge`. La prevencion requiere monitorear el tamano de los documentos en las colecciones de crecimiento potencial.

## Nombres consistentes

La consistencia en los nombres de campos y colecciones reduce la friccion cognitiva y previene errores. Las convenciones recomendadas son:

- **snake_case** para campos: `fecha_creacion`, `nombre_usuario`, `total_pedido`.
- **camelCase** para metodos y variables en el codigo de aplicacion, pero snake_case en la base de datos.
- Nombres de colecciones en plural: `usuarios`, `pedidos`, `productos`.
- Prefijos para campos que requieren contexto: `dir_envio_calle`, `dir_envio_ciudad`.

La consistencia es especialmente importante en los campos que se usan en queries y ordenaciones. Un campo escrito como `creadoEn` en unos documentos y `created_at` en otros obliga a consultas con `$or` o a migraciones correctivas.

## Validacion con $jsonSchema

MongoDB permite definir un esquema de validacion a nivel de coleccion mediante `$jsonSchema`. Aunque la base de datos no exige un esquema fijo, la validacion evita que datos mal formados contaminen la coleccion y traslada parte de la logica de integridad a la capa de persistencia.

```javascript
db.createCollection("usuarios", {
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: ["nombre", "email", "fecha_registro"],
         properties: {
            nombre: {
               bsonType: "string",
               description: "nombre completo del usuario"
            },
            email: {
               bsonType: "string",
               pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
            },
            edad: {
               bsonType: "int",
               minimum: 0,
               maximum: 150
            },
            direcciones: {
               bsonType: "array",
               items: {
                  bsonType: "object",
                  required: ["calle", "ciudad"],
                  properties: {
                     calle: { bsonType: "string" },
                     ciudad: { bsonType: "string" }
                  }
               }
            }
         }
      }
   }
})
```

La validacion puede configurarse con `validationAction: "warn"` para registrar las violaciones sin rechazar la escritura, lo que resulta util durante migraciones o cambios de esquema. Una vez validado el comportamiento, se cambia a `"error"`.

> `$jsonSchema` valida en el servidor, pero no reemplaza la validacion en la capa de aplicacion. La doble validacion (aplicacion + base de datos) protege contra clientes desactualizados, errores de programacion y accesos directos a la base de datos.

## Evitar colecciones grandes

Una coleccion con cientos de millones de documentos presenta desafios operativos: las operaciones de mantenimiento (como la construccion de indices o `compact`) tardan horas, el rendimiento de las consultas se degrada incluso con indices optimos, y la restauracion desde backup se vuelve lenta.

Para mitigar estos problemas existen dos estrategias principales:

- **Sharding**: distribuye los documentos de una coleccion entre multiples servidores (shards) usando una clave de fragmentacion (`shard key`). Cada shard gestiona un subconjunto de los datos, lo que permite que las consultas se ejecuten en paralelo. El sharding es la solucion definitiva para colecciones que superan la capacidad de una maquina individual, pero introduce complejidad operativa: hay que elegir bien la shard key, balancear los chunks y monitorear la distribucion.

- **Bucket pattern**: agrupa documentos relacionados en un contenedor temporal. Es especialmente util para datos de series temporales (logs, lecturas de sensores). En lugar de un documento por lectura, se almacena un array de lecturas dentro de una ventana de tiempo (por ejemplo, una hora). Esto reduce el numero de documentos y mejora la eficiencia de indices y operaciones de lectura.

```javascript
// En lugar de:
{ sensor: "temp-01", valor: 22.5, timestamp: ISODate("2025-01-01T00:00:00Z") }
{ sensor: "temp-01", valor: 22.7, timestamp: ISODate("2025-01-01T00:01:00Z") }

// Bucket por hora:
{
   sensor: "temp-01",
   hora: ISODate("2025-01-01T00:00:00Z"),
   lecturas: [
      { minuto: 0, valor: 22.5 },
      { minuto: 1, valor: 22.7 }
   ]
}
```

El bucket pattern reduce el recuento de documentos en varios ordenes de magnitud y permite usar indices TTL para expirar automaticamente los buckets antiguos.

