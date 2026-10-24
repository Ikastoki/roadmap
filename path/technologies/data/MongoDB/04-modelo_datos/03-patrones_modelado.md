# Patrones de modelado

Modelar datos en MongoDB requiere un cambio de mentalidad respecto al diseno relacional. La normalizacion y la descomposicion en tablas dan paso a un enfoque donde el patron de acceso de la aplicacion dicta la estructura de los documentos. No existe un modelo universalmente correcto: cada diseno es un compromiso entre eficiencia de lectura, flexibilidad de escritura y consistencia de los datos.

## Embedding (datos anidados)

El embedding consiste en incrustar subdocumentos directamente dentro del documento padre. Es el patron mas caracteristico de MongoDB y el que mayor rendimiento ofrece para lecturas, porque un unico documento contiene toda la informacion necesaria sin necesidad de joins.

Este patron es optimo cuando la relacion es **1:N pequena**, los datos hijos se acceden siempre junto con el padre, y las actualizaciones son atomicas a nivel de documento:

```javascript
{
  _id: ObjectId("..."),
  nombre: "Maria Lopez",
  direcciones: [
    { tipo: "casa", calle: "Av. siempre viva 123", ciudad: "Madrid" },
    { tipo: "trabajo", calle: "Calle mayor 45", ciudad: "Madrid" }
  ]
}
```

Las ventajas del embedding son:
- Una sola consulta obtiene todos los datos relacionados.
- No existe riesgo de datos huerfanos (el padre y los hijos se insertan y eliminan juntos).
- Actualizaciones atomicas sobre el conjunto completo.

Las desventajas aparecen cuando:
- El array embebido crece sin limite (riesgo de superar los 16 MB por documento).
- Se necesita acceder al subdocumento de forma independiente sin el padre.
- Varios documentos padre comparten el mismo subdocumento (duplicacion de datos).

> La regla practica es: si el array embebido puede crecer sin cota conocida, no uses embedding. Los casos tipicos donde el embedding funciona bien son items de factura, direcciones de envio, comentarios recientes o metadatos de sesion.

## Referencing (referencias)

El referencing almacena el identificador del documento relacionado en lugar del documento completo. Es el equivalente a las claves foraneas en SQL, pero sin enforcement automatico: la consistencia se gestiona desde la aplicacion.

```javascript
// Documento de usuario
{
  _id: ObjectId("user1"),
  nombre: "Ana Garcia"
}

// Documento de pedido con referencia al usuario
{
  _id: ObjectId("order1"),
  usuario_id: ObjectId("user1"),
  total: 150.00,
  items: [...]
}
```

Este patron es la opcion correcta cuando:
- La relacion es **N:M** (muchos a muchos): un producto puede estar en muchos pedidos, un pedido tiene muchos productos.
- Los datos relacionados son **grandes** y se acceden con frecuencia de forma independiente.
- Se necesita **integridad referencial** gestionada externamente.
- Las escrituras son mas frecuentes que las lecturas y no se quiere pagar el coste de actualizar datos duplicados.

La principal desventaja es la necesidad de multiples consultas (o un `$lookup`) para reunir los datos, lo que anyade latencia y complejidad.

## Subset pattern

Cuando un documento principal tiene una relacion 1:N con datos historicos que crecen sin limite, pero solo los datos mas recientes son relevantes para la mayoria de las consultas, el patron subset extrae los N elementos mas accedidos y los embebe en el documento padre, mientras el resto permanece en una coleccion separada.

```javascript
// Documento de usuario con las 5 reviews mas recientes
{
  _id: ObjectId("user1"),
  nombre: "Carlos Ruiz",
  reviews_recientes: [
    { producto: "Laptop X", puntuacion: 5, fecha: ISODate("2026-07-20") },
    { producto: "Teclado Y", puntuacion: 4, fecha: ISODate("2026-07-18") }
  ]
}

// Coleccion separada con todas las reviews
{
  usuario_id: ObjectId("user1"),
  producto: "Monitor Z",
  puntuacion: 3,
  fecha: ISODate("2023-05-10")
}
```

Este patron reduce drasticamente el tamano de los documentos mas consultados, mejorando el rendimiento de las lecturas sin perder acceso al historial completo.

## Bucket pattern

El bucket pattern agrupa documentos relacionados por un periodo de tiempo fijo dentro de un mismo documento, convirtiendo una coleccion de muchos documentos pequenos en una de pocos documentos de tamano medio. Es especialmente util para datos de series temporales, logs y lecturas de sensores.

```javascript
// Sin bucket: un documento por cada lectura
{ sensor: "temp-01", valor: 23.5, timestamp: ISODate("2026-07-28T10:00:00Z") }
{ sensor: "temp-01", valor: 23.7, timestamp: ISODate("2026-07-28T10:01:00Z") }
{ sensor: "temp-01", valor: 23.6, timestamp: ISODate("2026-07-28T10:02:00Z") }

// Con bucket: un documento por hora con array de lecturas
{
  sensor: "temp-01",
  hora: ISODate("2026-07-28T10:00:00Z"),
  lecturas: [
    { t: ISODate("2026-07-28T10:00:00Z"), v: 23.5 },
    { t: ISODate("2026-07-28T10:01:00Z"), v: 23.7 },
    { t: ISODate("2026-07-28T10:02:00Z"), v: 23.6 }
  ],
  total_lecturas: 60,
  avg: 23.6,
  min: 23.1,
  max: 24.0
}
```

Las ventajas del bucket pattern son:
- Reduccion del numero total de documentos (menos presion sobre el indice de `_id` y el catologo de colecciones).
- Precomputacion de agregados como media, minimo y maximo dentro del bucket.
- Compresion natural de datos historicos.

## Schema versioning

El schema versioning anyade un campo `schemaVersion` a los documentos para identificar la version del modelo que siguen. Esto permite que distintas versiones del modelo de datos coexistan en la misma coleccion mientras la aplicacion migra gradualmente los documentos antiguos.

```javascript
// Documento v1
{ _id: 1, schemaVersion: 1, nombre: "Juan", telefono: "123456789" }

// Documento v2 (telefono dividido en prefijo y numero)
{ _id: 2, schemaVersion: 2, nombre: "Ana", prefijo: "+34", telefono: "987654321" }
```

El proceso de migracion tipico consiste en:
1. Anyadir el campo `schemaVersion` a los documentos existentes (valor 1 por defecto).
2. Modificar la aplicacion para escribir documentos v2.
3. Ejecutar un script batch que actualice documentos v1 a v2 en segundo plano.
4. Cuando todos los documentos esten migrados, eliminar el codigo heredado.

Este patron permite migraciones progresivas sin downtime, algo imposible en bases de datos relacionales donde ALTER TABLE bloquea la tabla.

## Computed pattern (valores calculados)

El computed pattern almacena en el documento valores que podrian calcularse a partir de otros datos, pero cuyo calculo en tiempo de lectura resultaria demasiado costoso. Es especialmente util en agregaciones que se repiten con frecuencia.

```javascript
// Documento de pedido con total precalculado
{
  _id: ObjectId("order1"),
  items: [
    { producto: "A", precio: 10, cantidad: 2 },
    { producto: "B", precio: 15, cantidad: 1 }
  ],
  total: 35  // valor calculado: 10*2 + 15*1
}
```

Los casos de uso mas habituales son:
- **Total de factura**: precalculado al insertar o modificar el pedido.
- **Contador de relaciones**: numero de comentarios de un post, numero de seguidores de un usuario.
- **Promedios y agregados**: puntuacion media de un producto actualizada tras cada review.
- **Tokens de busqueda**: campos concatenados y normalizados para busquedas de texto.

La desventaja es la necesidad de mantener la consistencia: cuando cambian los datos origen, el valor calculado debe actualizarse. MongoDB no tiene disparadores ni vistas materializadas; la responsabilidad recae enteramente en la aplicacion.

