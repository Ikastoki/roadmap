# Etapas basicas del aggregation pipeline

Las etapas basicas constituyen los bloques fundamentales del aggregation framework. La mayoria de los pipelines de agregacion en produccion utilizan una combinacion de estas etapas para filtrar, agrupar, transformar y ordenar documentos. Cada etapa esta optimizada para un proposito especifico y, cuando se combinan correctamente, permiten construir analisis que equivaldrian a consultas SQL complejas con GROUP BY, JOIN y HAVING.

## $match

Filtra documentos del pipeline de la misma forma que la clausula `find()`: solo los documentos que cumplen la condicion especificada pasan a la siguiente etapa. La sintaxis es identica a la de los filtros de `find()`.

```javascript
// Filtrar pedidos del ultimo trimestre de 2025
db.pedidos.aggregate([
    { $match: { fecha: { $gte: ISODate("2025-10-01"), $lt: ISODate("2026-01-01") } } }
])
```

Cuando `$match` es la primera etapa del pipeline, MongoDB puede utilizar indices de la coleccion para resolver el filtro, exactamente igual que en una operacion `find()`. Si `$match` aparece en posiciones posteriores, actua sobre documentos ya transformados y no puede aprovechar indices.

```javascript
// $match en primera etapa = usa indice en 'status'
db.ordenes.aggregate([
    { $match: { status: "completada" } },
    { $group: { _id: "$cliente_id", total: { $sum: "$total" } } }
])
```

> Para maximizar el rendimiento, colocar `$match` como primera etapa del pipeline siempre que sea posible. Un `$match` en medio del pipeline fuerza un escaneo completo de los documentos resultantes de la etapa anterior.

## $group

Agrupa documentos por un campo o expresion especificada en `_id` y aplica acumuladores a los documentos de cada grupo. Es la etapa equivalente a `GROUP BY` de SQL y el corazon de las agregaciones.

```javascript
// Agrupar ventas por categoria calculando total y promedio
db.ventas.aggregate([
    { $group: {
        _id: "$categoria",
        total_ventas: { $sum: "$importe" },
        promedio: { $avg: "$importe" },
        minimo: { $min: "$importe" },
        maximo: { $max: "$importe" },
        productos: { $push: "$producto" },
        primer_producto: { $first: "$producto" }
    }}
])
```

**Acumuladores comunes en $group:**

| Acumulador | Descripcion |
|---|---|
| `$sum` | Suma de valores numericos |
| `$avg` | Promedio de valores numericos |
| `$min` | Valor minimo |
| `$max` | Valor maximo |
| `$push` | Array con todos los valores del campo |
| `$first` | Primer valor del grupo (requiere `$sort` previo) |
| `$last` | Ultimo valor del grupo (requiere `$sort` previo) |
| `$addToSet` | Array con valores unicos del campo |

El campo `_id` en `$group` define la clave de agrupacion. Puede ser un solo campo (`"$categoria"`) o un documento compuesto para agrupar por multiples criterios:

```javascript
// Agrupar por anyo y mes
db.ventas.aggregate([
    { $group: {
        _id: {
            anyo: { $year: "$fecha" },
            mes: { $month: "$fecha" }
        },
        total: { $sum: "$importe" }
    }}
])
```

Los acumuladores `$first` y `$last` solo tienen sentido cuando los documentos de entrada estan ordenados. Si se usa `$sort` antes de `$group`, `$first` devuelve el primer documento segun el orden establecido y `$last` el ultimo.

## $project

Selecciona, excluye, renombra o calcula campos en los documentos que pasan por el pipeline. Funciona de manera similar a la proyeccion de `find()`, pero permite crear nuevos campos basados en expresiones y operadores.

```javascript
// Proyeccion selectiva con campos caculados
db.pedidos.aggregate([
    { $project: {
        _id: 0,
        cliente: 1,
        importe_total: { $multiply: ["$precio_unitario", "$cantidad"] },
        fecha_pedido: "$fecha",
        con_iva: { $multiply: ["$importe", 1.21] }
    }}
])
```

El valor `1` incluye el campo en la salida, `0` lo excluye. Cuando se usan expresiones, el resultado se asigna al nombre del campo. `$project` puede reestructurar completamente la forma del documento:

```javascript
// Reestructurar documento anidado
db.pedidos.aggregate([
    { $project: {
        direccion_entrega: {
            calle: "$direccion.calle",
            ciudad: "$direccion.ciudad",
            codigo_postal: "$direccion.cp"
        }
    }}
])
```

> `$project` solo puede incluir o excluir campos, pero no ambas cosas simultaneamente, excepto para `_id`. Si se incluyen campos explicitamente, todos los no listados se excluyen. Si se excluyen campos, todos los no listados se incluyen.

## $sort

Ordena los documentos ascendente (1) o descendentemente (-1) segun uno o varios campos.

```javascript
// Ordenar por precio descendente y nombre ascendente
db.productos.aggregate([
    { $sort: { precio: -1, nombre: 1 } }
])
```

Cuando `$sort` aparece al inicio del pipeline, MongoDB puede usar un indice para resolver la ordenacion sin necesidad de consumir memoria. Si el `$sort` ocurre en medio del pipeline, MongoDB debe ordenar los documentos en memoria, lo que puede ser costoso con grandes volumenes.

```javascript
// $sort despues de $group: ordenacion en memoria
db.ventas.aggregate([
    { $group: { _id: "$categoria", total: { $sum: "$importe" } } },
    { $sort: { total: -1 } }
])
```

Combinado con `$limit`, MongoDB optimiza automaticamente la ordenacion manteniendo solo los `n` mejores elementos en un buffer de memoria (top-k sort).

## $limit y $skip

Controlan la paginacion de los resultados del pipeline. `$limit` restringe el numero de documentos que pasan a la siguiente etapa, mientras que `$skip` descarta un numero especifico de documentos al inicio.

```javascript
// Paginacion: saltar 20 documentos y tomar 10
db.productos.aggregate([
    { $sort: { precio: -1 } },
    { $skip: 20 },
    { $limit: 10 }
])
```

`$skip` es inherentemente ineficiente para valores grandes porque MongoDB debe iterar sobre todos los documentos descartados. Para paginacion profunda es preferible usar paginacion basada en cursor (seek method) en lugar de `$skip`.

## $unwind

Descompone cada elemento de un array en un documento independiente. Si un documento tiene un array con tres elementos, `$unwind` produce tres documentos, cada uno con el mismo contenido del documento original pero con el array reemplazado por un elemento individual.

```javascript
// Expandir array de lineas de pedido
db.pedidos.aggregate([
    { $unwind: { path: "$lineas", preserveNullAndEmptyArrays: true } }
])
```

El parametro `preserveNullAndEmptyArrays` controla si los documentos sin array (o con array vacio) se conservan en el pipeline. Por defecto es `false`, lo que significa que esos documentos se pierden.

`$unwind` suele utilizarse antes de `$group` cuando se necesita agregar datos que estan dentro de arrays. Tambien es comun detras de `$lookup` para expandir los resultados de la union.

```javascript
// Calcular total por producto a partir de lineas de pedido
db.pedidos.aggregate([
    { $unwind: "$lineas" },
    { $group: {
        _id: "$lineas.producto_id",
        total_vendido: { $sum: "$lineas.cantidad" }
    }}
])
```

## $count

Devuelve un unico documento con el numero total de documentos que han llegado a esa etapa del pipeline.

```javascript
db.ventas.aggregate([
    { $match: { importe: { $gt: 100 } } },
    { $count: "ventas_grandes" }
])
// Resultado: { "ventas_grandes": 42 }
```

El nombre del campo en el documento de salida se especifica como argumento.

## $lookup

Realiza un LEFT JOIN entre la coleccion actual y otra coleccion externa. Es la etapa que permite cruzar datos entre colecciones, equivalente a `JOIN` en SQL.

```javascript
// Lookup basico
db.pedidos.aggregate([
    { $lookup: {
        from: "clientes",
        localField: "cliente_id",
        foreignField: "_id",
        as: "cliente"
    }}
])
```

**Parametros:**

- `from`: coleccion destino
- `localField`: campo en la coleccion actual
- `foreignField`: campo en la coleccion destino
- `as`: nombre del array donde se almacenan los documentos coincidentes

El resultado es un array de documentos en el campo `as`. Normalmente se combina con `$unwind` para aplanar el resultado si se espera una sola coincidencia.

```javascript
// Lookup con unwind para desanidar el resultado
db.pedidos.aggregate([
    { $lookup: {
        from: "clientes",
        localField: "cliente_id",
        foreignField: "_id",
        as: "cliente"
    }},
    { $unwind: { path: "$cliente", preserveNullAndEmptyArrays: true } }
])
```

### Lookup con pipeline

La forma mas potente de `$lookup` permite especificar un pipeline completo sobre la coleccion externa, incluyendo `$match`, `$project`, `$sort` y otras etapas.

```javascript
// Lookup con pipeline: solo pedidos activos del cliente
db.clientes.aggregate([
    { $lookup: {
        from: "pedidos",
        let: { cliente_id: "$_id" },
        pipeline: [
            { $match: {
                $expr: { $eq: ["$cliente_id", "$$cliente_id"] },
                status: "activo"
            }},
            { $sort: { fecha: -1 } },
            { $limit: 5 }
        ],
        as: "ultimos_pedidos"
    }}
])
```

Las variables definidas en `let` se referencian con `$$` dentro del pipeline. Esto permite condiciones de join mas complejas que la simple igualdad de campos.

> `$lookup` es una operacion costosa. Cuando sea posible, disenar el esquema para evitar joins mediante embedding de datos. Si el `$lookup` es inevitable, usar indices en los campos involucrados en la condicion de join.

