# Relaciones entre documentos

MongoDB no soporta joins en el sentido relacional clasico, pero proporciona mecanismos para relacionar documentos entre colecciones. La decision fundamental en el modelado es elegir entre representar las relaciones mediante embedding o referencing, y cuando usar referencing, saber aplicar las operaciones del pipeline de agregacion que reconstruyen los datos relacionados.

## Relacion 1:1

La relacion uno a uno es la mas simple de modelar. La pregunta relevante es si los dos conjuntos de atributos se acceden siempre juntos o no.

Cuando los datos se acceden siempre juntos, el embedding es la opcion natural:

```javascript
{
  _id: ObjectId("user1"),
  nombre: "Laura",
  perfil: {
    biografia: "Ingeniera de software",
    avatar_url: "/images/avatar1.png",
    fecha_registro: ISODate("2024-01-15")
  }
}
```

Cuando los datos se acceden por separado o uno de los lados es muy grande, el referencing es preferible:

```javascript
// Coleccion usuarios
{ _id: ObjectId("user1"), nombre: "Laura" }

// Coleccion perfiles (referencia 1:1)
{ _id: ObjectId("profile1"), usuario_id: ObjectId("user1"), biografia: "Ingeniera de software" }
```

## Relacion 1:N

La relacion uno a muchos es la mas frecuente en cualquier sistema. Un usuario tiene muchos pedidos, un producto tiene muchas reviews, un blog tiene muchos posts.

La decision entre embedding y referencing depende del tamano esperado del lado "muchos":

**Embedding** cuando la cardinalidad es pequena y acotada (menos de unos cientos de elementos):

```javascript
{
  _id: ObjectId("user1"),
  nombre: "Pedro",
  direcciones: [
    { calle: "Calle Sol 5", ciudad: "Valencia" },
    { calle: "Av. Mar 12", ciudad: "Barcelona" }
  ]
}
```

**Referencing** cuando el array puede crecer sin limite predecible:

```javascript
// Coleccion usuarios
{ _id: ObjectId("user1"), nombre: "Pedro" }

// Coleccion pedidos (referencia 1:N)
{ _id: ObjectId("order1"), usuario_id: ObjectId("user1"), total: 99.99, fecha: ISODate("2026-07-01") }
{ _id: ObjectId("order2"), usuario_id: ObjectId("user1"), total: 150.00, fecha: ISODate("2026-07-15") }
```

El referencing en 1:N tambien permite usar un patron de **caja fuerte** cuando el documento padre necesita acceso rapido a una lista de IDs hijos sin cargar los documentos completos:

```javascript
{
  _id: ObjectId("user1"),
  nombre: "Pedro",
  pedidos_ids: [ObjectId("order1"), ObjectId("order2")]
}
```

## Relacion N:M

Muchos a muchos es la relacion que mas claramente diferencia a MongoDB de SQL. Sin tablas intermedias con claves foraneas compuestas, la representacion depende de como se acceden los datos.

Cuando el acceso es simetrico (se accede tanto desde A a B como desde B a A con frecuencia similar), lo habitual es almacenar arrays de referencias en ambos lados:

```javascript
// Coleccion estudiantes
{
  _id: ObjectId("student1"),
  nombre: "Elena",
  cursos_ids: [ObjectId("course1"), ObjectId("course2")]
}

// Coleccion cursos
{
  _id: ObjectId("course1"),
  nombre: "Matematicas",
  estudiantes_ids: [ObjectId("student1"), ObjectId("student3")]
}
```

Cuando la relacion tiene atributos propios (una nota en un curso, una fecha de inscripcion), se necesita una tercera coleccion que actua como documento de union, similar a la tabla intermedia en SQL:

```javascript
// Coleccion inscripciones (relacion N:M con atributos)
{
  _id: ObjectId("enroll1"),
  estudiante_id: ObjectId("student1"),
  curso_id: ObjectId("course1"),
  nota: 8.5,
  fecha_inscripcion: ISODate("2026-01-10"),
  estado: "activo"
}
```

## Operador $lookup

`$lookup` es la operacion del pipeline de agregacion que realiza un left outer join entre colecciones. Se introduce en MongoDB 3.2 y se extiende significativamente en MongoDB 3.6 con soporte para condiciones mas complejas.

### Sintaxis basica

```javascript
db.pedidos.aggregate([
  {
    $lookup: {
      from: "usuarios",
      localField: "usuario_id",
      foreignField: "_id",
      as: "usuario"
    }
  }
]);
```

El resultado anyade un array `usuario` a cada pedido con los documentos de la coleccion `usuarios` que coinciden en la condicion. Dado que `_id` es unico en `usuarios`, el array tendra un unico elemento. Se puede desempaquetar con `$unwind`.

### Sintaxis avanzada (MongoDB 3.6+)

La sintaxis con `let` y `pipeline` permite condiciones arbitrarias, incluyendo joins compuestos por multiples campos y filtros previos:

```javascript
db.pedidos.aggregate([
  {
    $lookup: {
      from: "productos",
      let: { pedido_items: "$items" },
      pipeline: [
        { $match: { $expr: { $in: ["$_id", "$$pedido_items.producto_id"] } } },
        { $project: { nombre: 1, precio: 1 } }
      ],
      as: "productos_detalle"
    }
  }
]);
```

Este enfoque es mas flexible pero menos eficiente que la sintaxis basica, porque ejecuta un subpipeline completo para cada documento del lote.

### Rendimiento de $lookup

`$lookup` no tiene el mismo rendimiento que un JOIN en SQL. Cada operacion implica:
- Un escaneo o busqueda en la coleccion destino por cada documento del lote.
- La transferencia de datos entre el motor de agregacion y el almacenamiento.
- La materializacion de los resultados en memoria.

Para que `$lookup` sea eficiente, la coleccion destino debe tener un indice sobre el campo de join. Sin indice, `$lookup` realiza un collscan completo por cada documento del origen, lo que puede degradar el rendimiento drasticamente.

> `$lookup` es una herramienta poderosa pero no debe abusarse. Cada join en MongoDB cuesta significativamente mas que en SQL. El diseno de documentos debe priorizar la denormalizacion y el embedding, dejando `$lookup` para casos donde la normalizacion es inevitable.

## Operador $unwind

`$unwind` descompone un array en multiples documentos, uno por cada elemento del array. Es el complemento natural de `$lookup` cuando se espera un unico documento relacionado y se quiere evitar trabajar con arrays:

```javascript
db.pedidos.aggregate([
  {
    $lookup: {
      from: "usuarios",
      localField: "usuario_id",
      foreignField: "_id",
      as: "usuario"
    }
  },
  { $unwind: "$usuario" },
  { $project: { "usuario.nombre": 1, total: 1 } }
]);
```

Sin `$unwind`, el campo `usuario` es un array. Con `$unwind`, cada pedido genera un documento con `usuario` como objeto en lugar de array. Es importante tener en cuenta que si el array esta vacio, `$unwind` elimina el documento por defecto. Para preservarlo se usa `{ $unwind: { path: "$usuario", preserveNullAndEmptyArrays: true } }`.

### Combinacion $lookup + $unwind

La combinacion de ambos operadores es el patron estandar para reconstruir datos normalizados en MongoDB. Sin embargo, cada `$lookup` introduce latencia, y un pipeline con multiples `$lookup` puede volverse significativamente mas lento que una consulta equivalente en SQL con varios JOINs.

```javascript
db.pedidos.aggregate([
  { $lookup: { from: "usuarios", localField: "usuario_id", foreignField: "_id", as: "usuario" } },
  { $unwind: "$usuario" },
  { $lookup: { from: "productos", localField: "items.producto_id", foreignField: "_id", as: "productos" } },
  { $unwind: "$productos" }
]);
```

Cada operacion de `$unwind` multiplica el numero de documentos del pipeline. Un pedido con 10 items y un `$lookup` a productos seguiado de `$unwind` genera 10 documentos intermedios. Con dos `$lookup` anidados, el crecimiento es exponencial. Por eso es recomendable usar `$group` para re-agrupar despues de `$unwind` cuando sea necesario, o reconsiderar si el modelo no deberia usar embedding en lugar de tantos joins.

