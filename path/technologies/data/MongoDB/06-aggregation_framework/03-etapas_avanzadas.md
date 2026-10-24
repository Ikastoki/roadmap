# Etapas avanzadas del aggregation pipeline

Mas alla de las etapas basicas, el aggregation framework ofrece etapas especializadas para analisis multidimensionales, busquedas jerarquicas, relleno de series temporales y persistencia de resultados. Estas etapas permiten construir pipelines que resuelven problemas analiticos complejos directamente en la base de datos, sin necesidad de procesamiento externo.

## $facet

Ejecuta multiples pipelines de agregacion en paralelo sobre el mismo conjunto de documentos de entrada y devuelve los resultados en un unico documento. Cada subpipeline especificado en `$facet` recibe la misma entrada y produce su propio array de resultados.

```javascript
db.ventas.aggregate([
    { $match: { fecha: { $gte: ISODate("2025-01-01") } } },
    { $facet: {
        totales: [
            { $group: { _id: null, total: { $sum: "$importe" }, promedio: { $avg: "$importe" } } }
        ],
        por_categoria: [
            { $group: { _id: "$categoria", total: { $sum: "$importe" } } },
            { $sort: { total: -1 } }
        ],
        top_productos: [
            { $group: { _id: "$producto", unidades: { $sum: "$cantidad" } } },
            { $sort: { unidades: -1 } },
            { $limit: 5 }
        ]
    }}
])
```

**Salida tipica:**

```json
{
    "totales": [ { "_id": null, "total": 145000, "promedio": 234.5 } ],
    "por_categoria": [ { "_id": "electronica", "total": 65000 }, ... ],
    "top_productos": [ { "_id": "auriculares", "unidades": 1200 }, ... ]
}
```

`$facet` es util para construir dashboards y reportes donde se necesitan multiples metricas sobre el mismo conjunto de datos sin tener que ejecutar consultas separadas. Cada subpipeline puede incluir sus propias etapas de `$match`, `$group`, `$sort`, `$limit`, etc.

> Limitacion: `$facet` no puede contener etapas que escriban datos (`$out`, `$merge`) ni `$facet` anidados.

## $bucket y $bucketAuto

Clasifican documentos en grupos (buckets) segun rangos de valores. Son equivalentes a `CASE WHEN ... THEN ... END` con rangos en SQL.

### $bucket

Define los limites de los buckets explicitamente.

```javascript
// Agrupar productos por rango de precio
db.productos.aggregate([
    { $bucket: {
        groupBy: "$precio",
        boundaries: [0, 50, 100, 200, 500, 1000],
        default: "otros",
        output: {
            count: { $sum: 1 },
            promedio_precio: { $avg: "$precio" }
        }
    }}
])
```

- `groupBy`: expresion que determina el valor de clasificacion
- `boundaries`: array ordenado de limites (cada bucket cubre [limite, limite_siguiente))
- `default`: bucket para valores fuera de los limites
- `output`: acumuladores para cada bucket

### $bucketAuto

Determina los limites de los buckets automaticamente para distribuir los documentos de forma uniforme.

```javascript
// 5 buckets automaticos con distribucion uniforme
db.productos.aggregate([
    { $bucketAuto: {
        groupBy: "$precio",
        buckets: 5,
        output: {
            count: { $sum: 1 },
            precio_medio: { $avg: "$precio" }
        }
    }}
])
```

`$bucketAuto` utiliza el algoritmo de cuantiles para determinar los limites, asegurando que cada bucket contenga aproximadamente el mismo numero de documentos. El parametro `granularity` permite controlar la precision de los limites (R5, R10, R20, etc.).

## $sortByCount

Combina `$group` y `$sort` para contar y ordenar por frecuencia de ocurrencia de un campo. Es un atajo para el patron tipico de agrupar por campo, contar y ordenar descendente.

```javascript
// Categorias mas frecuentes
db.ventas.aggregate([
    { $sortByCount: "$categoria" }
])

// Equivalente a:
db.ventas.aggregate([
    { $group: { _id: "$categoria", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
])
```

El resultado incluye `_id` (el valor del campo) y `count` (el numero de ocurrencias).

## $graphLookup

Realiza una busqueda recursiva sobre una coleccion para resolver relaciones jerarquicas o de grafo. Es util para arboles de categorias, organigramas, rutas de navegacion y cualquier estructura donde los documentos se referencien entre si formando una jerarquia.

```javascript
// Buscar jerarquia de categorias recursivamente
db.categorias.aggregate([
    { $match: { nombre: "electronica" } },
    { $graphLookup: {
        from: "categorias",
        startWith: "$_id",
        connectFromField: "_id",
        connectToField: "padre_id",
        as: "subcategorias",
        maxDepth: 5,
        depthField: "nivel"
    }}
])
```

**Parametros:**

- `from`: coleccion destino
- `startWith`: valor inicial de la busqueda
- `connectFromField`: campo del documento actual que se enlaza
- `connectToField`: campo del documento destino que recibe el enlace
- `as`: nombre del array de resultados
- `maxDepth`: profundidad maxima de recursion
- `depthField`: campo opcional para almacenar el nivel de profundidad
- `restrictSearchWithMatch`: filtro opcional para restringir la busqueda

```javascript
// Organigrama: buscar todos los subordinados de un manager
db.empleados.aggregate([
    { $match: { nombre: "Maria Garcia" } },
    { $graphLookup: {
        from: "empleados",
        startWith: "$_id",
        connectFromField: "_id",
        connectToField: "manager_id",
        as: "equipo",
        maxDepth: 10,
        depthField: "nivel_jerarquico"
    }}
])
```

> `$graphLookup` es significativamente mas costoso que un `$lookup` normal. Limitar la profundidad con `maxDepth` y usar indices en `connectToField` para mitigar el impacto.

## $out y $merge

Persisten los resultados del pipeline en una coleccion. La diferencia fundamental es que `$out` reemplaza completamente la coleccion de salida, mientras que `$merge` puede combinar los resultados con datos existentes.

### $out

Escribe los resultados en una coleccion nueva, reemplazandola si ya existe. Debe ser la ultima etapa del pipeline.

```javascript
// Almacenar resultado de agregacion en nueva coleccion
db.ventas.aggregate([
    { $group: { _id: "$categoria", total: { $sum: "$importe" } } },
    { $out: "resumen_ventas" }
])
```

### $merge

Permite mayor control sobre como se combinan los resultados con la coleccion destino. Soporta diferentes acciones segun si el documento coincide o no.

```javascript
// Fusionar resultados existentes
db.ventas.aggregate([
    { $group: { _id: "$categoria", total: { $sum: "$importe" } } },
    { $merge: {
        into: "resumen_ventas",
        on: "_id",
        whenMatched: "replace",
        whenNotMatched: "insert"
    }}
])
```

**Opciones de `whenMatched`:** `replace` (reemplazar), `merge` (fusionar campos), `keepExisting` (conservar), `fail` (error).

**Opciones de `whenNotMatched`:** `insert` (insertar), `discard` (descartar), `fail` (error).

`$merge` puede escribir en una base de datos diferente si se especifica `into` como `{ db: "otra_bd", coll: "coleccion" }`.

## $densify

Rellena gaps en series temporales creando documentos intermedios para puntos temporales faltantes. Es esencial para analisis de series temporales donde los datos no estan uniformemente espaciados.

```javascript
// Densificar por hora, rellenando gaps de 1 hora
db.sensores.aggregate([
    { $match: { sensor_id: "temp_01" } },
    { $densify: {
        field: "timestamp",
        range: {
            step: 1,
            unit: "hour",
            bounds: [ISODate("2025-01-01"), ISODate("2025-01-02")]
        }
    }}
])
```

**Parametros:**

- `field`: campo de fecha sobre el que densificar
- `range.step`: tamano del paso
- `range.unit`: unidad del paso (year, month, week, day, hour, minute, second)
- `range.bounds`: limites del rango a densificar (array de dos fechas)
- `partitionByFields`: campos opcionales para densificar por particion

Los documentos generados por `$densify` contienen solo el campo especificado y cualquier campo de particion; los demas campos se pierden.

## $fill

Rellena valores nulos o faltantes en los documentos usando diferentes estrategias: valor constante, valor del documento anterior o valor del documento siguiente.

```javascript
// Rellenar valores nulos con diferentes estrategias
db.sensores.aggregate([
    { $sort: { timestamp: 1 } },
    { $fill: {
        output: {
            temperatura: { method: "locf" },
            humedad: { value: 50 }
        }
    }}
])
```

**Metodos de relleno:**

| Metodo | Descripcion |
|---|---|
| `"locf"` | Last Observation Carried Forward (valor anterior no nulo) |
| `"linear"` | Interpolacion lineal entre valores no nulos anteriores y posteriores |
| `value: <expr>` | Rellenar con un valor fijo |

```javascript
// Interpolacion lineal entre valores
db.sensores.aggregate([
    { $sort: { timestamp: 1 } },
    { $fill: {
        sortBy: { timestamp: 1 },
        output: {
            temperatura: { method: "linear" }
        }
    }}
])
```

`$fill` requiere que los documentos esten ordenados. Se puede especificar `sortBy` dentro de la etapa o usar un `$sort` previo.

