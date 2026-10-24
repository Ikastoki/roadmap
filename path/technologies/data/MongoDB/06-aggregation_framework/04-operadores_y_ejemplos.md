# Operadores y ejemplos del aggregation framework

Los operadores del aggregation framework permiten transformar, calcular y manipular valores dentro de las etapas del pipeline. Se dividen en categorias segun el tipo de dato sobre el que operan: estadisticos, array, string, condicionales, fecha y tipo. Combinados con las etapas del pipeline, estos operadores permiten construir practicamente cualquier logica de transformacion de datos directamente en MongoDB.

## Operadores estadisticos

Se utilizan principalmente dentro de `$group` y `$project` para calcular metricas agregadas sobre conjuntos de documentos.

```javascript
db.ventas.aggregate([
    { $group: {
        _id: "$categoria",
        total: { $sum: "$importe" },
        promedio: { $avg: "$importe" },
        minimo: { $min: "$importe" },
        maximo: { $max: "$importe" },
        desviacion_estandar: { $stdDevPop: "$importe" },
        desviacion_muestra: { $stdDevSamp: "$importe" }
    }}
])
```

| Operador | Descripcion |
|---|---|
| `$sum` | Suma de valores numericos o contador (`{ $sum: 1 }`) |
| `$avg` | Media aritmetica |
| `$min` | Valor minimo |
| `$max` | Valor maximo |
| `$stdDevPop` | Desviacion estandar poblacional |
| `$stdDevSamp` | Desviacion estandar muestral |
| `$first` | Primer valor del grupo (con `$sort` previo) |
| `$last` | Ultimo valor del grupo (con `$sort` previo) |
| `$push` | Array con todos los valores |
| `$addToSet` | Array con valores unicos |

La diferencia entre `$stdDevPop` y `$stdDevSamp` es la misma que en estadistica clasica: `$stdDevPop` divide por N (poblacion total), mientras que `$stdDevSamp` divide por N-1 (correccion para muestra).

```javascript
// Contar y listar valores unicos
db.pedidos.aggregate([
    { $group: {
        _id: "$cliente_id",
        total_pedidos: { $sum: 1 },
        importe_total: { $sum: "$importe" },
        categorias: { $addToSet: "$categoria" }
    }}
])
```

## Operadores de array

Permiten manipular y transformar arrays dentro de los documentos. Son fundamentales para trabajar con datos embebidos.

### $filter

Selecciona elementos de un array que cumplen una condicion.

```javascript
db.pedidos.aggregate([
    { $project: {
        lineas_caras: {
            $filter: {
                input: "$lineas",
                as: "linea",
                cond: { $gte: ["$$linea.precio", 100] }
            }
        }
    }}
])
```

### $map

Transforma cada elemento de un array aplicando una expresion.

```javascript
db.pedidos.aggregate([
    { $project: {
        lineas_con_iva: {
            $map: {
                input: "$lineas",
                as: "linea",
                in: {
                    producto: "$$linea.producto",
                    precio_con_iva: { $multiply: ["$$linea.precio", 1.21] }
                }
            }
        }
    }}
])
```

### $reduce

Reduce un array a un unico valor aplicando una operacion acumulativa.

```javascript
db.pedidos.aggregate([
    { $project: {
        total_carrito: {
            $reduce: {
                input: "$lineas",
                initialValue: 0,
                in: { $add: ["$$value", { $multiply: ["$$this.precio", "$$this.cantidad"] }] }
            }
        }
    }}
])
```

### $size y $slice

`$size` devuelve la longitud del array. `$slice` extrae un subconjunto de elementos.

```javascript
db.productos.aggregate([
    { $project: {
        nombre: 1,
        num_tags: { $size: "$tags" },
        primeros_tags: { $slice: ["$tags", 3] }
    }}
])
```

| Operador | Descripcion |
|---|---|
| `$arrayElemAt` | Elemento en un indice especifico |
| `$concatArrays` | Concatenar dos o mas arrays |
| `$isArray` | Devuelve true si el valor es un array |
| `$arrayToObject` | Convierte array de pares clave-valor a objeto |
| `$objectToArray` | Convierte objeto a array de pares clave-valor |
| `$indexOfArray` | Indice de la primera ocurrencia de un valor |
| `$reverseArray` | Invierte el orden del array |

## Operadores de string

Operan sobre cadenas de texto para limpieza, transformacion y busqueda.

```javascript
db.usuarios.aggregate([
    { $project: {
        nombre_completo: { $concat: ["$nombre", " ", "$apellido"] },
        nombre_mayusculas: { $toUpper: "$nombre" },
        iniciales: { $substrCP: ["$nombre", 0, 1] },
        email_limpio: { $trim: { input: "$email" } }
    }}
])
```

**Busqueda con regex:**

```javascript
db.articulos.aggregate([
    { $project: {
        contenido: 1,
        tiene_urgencia: {
            $regexMatch: {
                input: "$texto",
                regex: "urgente|critico|inmediato",
                options: "i"
            }
        },
        todas_coincidencias: {
            $regexFindAll: {
                input: "$texto",
                regex: "\\b\\w{6,}\\b"
            }
        }
    }}
])
```

| Operador | Descripcion |
|---|---|
| `$concat` | Concatenar strings |
| `$toUpper` / `$toLower` | Mayusculas / minusculas |
| `$substr` / `$substrCP` / `$substrBytes` | Subcadena desde posicion |
| `$trim` / `$ltrim` / `$rtrim` | Eliminar espacios |
| `$regexFind` / `$regexFindAll` / `$regexMatch` | Busqueda con expresion regular |
| `$replaceOne` / `$replaceAll` | Reemplazar ocurrencias |
| `$indexOfBytes` / `$indexOfCP` | Indice de primera ocurrencia |
| `$strLenCP` / `$strLenBytes` | Longitud del string |

## Operadores condicionales

Implementan logica de decision dentro del pipeline: IF-THEN-ELSE, valores por defecto y switch-case.

### $cond

Evalua una condicion booleana y devuelve un valor u otro segun el resultado.

```javascript
db.pedidos.aggregate([
    { $project: {
        importe: 1,
        categoria_precio: {
            $cond: {
                if: { $gte: ["$importe", 1000] },
                then: "alta",
                else: "baja"
            }
        }
    }}
])

// Forma abreviada con array
db.pedidos.aggregate([
    { $project: {
        categoria_precio: {
            $cond: [{ $gte: ["$importe", 1000] }, "alta", "baja"]
        }
    }}
])
```

### $ifNull

Proporciona un valor por defecto si el campo es null o no existe.

```javascript
db.clientes.aggregate([
    { $project: {
        nombre: 1,
        telefono: { $ifNull: ["$telefono", "No disponible"] },
        email: { $ifNull: ["$email", "Sin email"] }
    }}
])
```

### $switch

Evalua multiples casos secuencialmente y ejecuta el primero que coincida (equivalente a CASE-WHEN).

```javascript
db.pedidos.aggregate([
    { $project: {
        importe: 1,
        segmento: {
            $switch: {
                branches: [
                    { case: { $gte: ["$importe", 5000] }, then: "Premium" },
                    { case: { $gte: ["$importe", 1000] }, then: "Gold" },
                    { case: { $gte: ["$importe", 100] }, then: "Standard" }
                ],
                default: "Basic"
            }
        }
    }}
])
```

## Operadores de fecha

Permiten extraer componentes de fechas, formatear, sumar intervalos y calcular diferencias.

```javascript
db.ventas.aggregate([
    { $project: {
        anyo: { $year: "$fecha" },
        mes: { $month: "$fecha" },
        dia: { $dayOfMonth: "$fecha" },
        hora: { $hour: "$fecha" },
        dia_semana: { $dayOfWeek: "$fecha" },
        fecha_formateada: {
            $dateToString: {
                format: "%Y-%m-%d %H:%M",
                date: "$fecha"
            }
        }
    }}
])
```

**Operaciones aritmeticas con fechas:**

```javascript
db.suscripciones.aggregate([
    { $project: {
        // Anyadir 30 dias a la fecha de inicio
        proxima_facturacion: { $dateAdd: { startDate: "$inicio", unit: "day", amount: 30 } },
        // Diferencia en dias entre dos fechas
        dias_activo: {
            $dateDiff: {
                startDate: "$inicio",
                endDate: "$$NOW",
                unit: "day"
            }
        },
        // Restar 1 anyo
        anyo_anterior: { $dateSubtract: { startDate: "$fecha", unit: "year", amount: 1 } }
    }}
])
```

| Operador | Descripcion |
|---|---|
| `$year`, `$month`, `$dayOfMonth`, `$dayOfWeek`, `$dayOfYear` | Extraer componentes |
| `$hour`, `$minute`, `$second`, `$millisecond` | Extraer componentes de hora |
| `$dateToString` | Formatear fecha a string |
| `$dateFromString` | Parsear string a fecha |
| `$dateFromParts` / `$dateToParts` | Construir/descomponer fecha |
| `$dateAdd` / `$dateSubtract` | Sumar/restar intervalos |
| `$dateDiff` | Diferencia entre fechas |
| `$toDate` | Convertir a tipo Date |
| `$$NOW` / `$$CLUSTER_TIME` | Fecha actual del sistema |

## Operadores de tipo

Convierten entre tipos de datos y permiten inspeccionar el tipo de un valor.

```javascript
db.productos.aggregate([
    { $project: {
        precio_entero: { $toInt: "$precio" },
        precio_texto: { $toString: "$precio" },
        activo_booleano: { $toBool: "$activo" },
        tipo_precio: { $type: "$precio" }
    }}
])
```

| Operador | Descripcion |
|---|---|
| `$toInt` / `$toLong` | Convertir a entero |
| `$toDouble` / `$toDecimal` | Convertir a decimal |
| `$toString` | Convertir a string |
| `$toBool` | Convertir a booleano |
| `$toDate` | Convertir a fecha |
| `$toObjectId` | Convertir a ObjectId |
| `$type` | Devuelve el tipo BSON del campo |
| `$convert` | Conversion generica con manejo de errores |

## Variables especiales

El aggregation framework expone variables especiales que referencian contextos especificos dentro del pipeline.

| Variable | Descripcion |
|---|---|
| `$$ROOT` | Referencia al documento original completo (antes de cualquier transformacion) |
| `$$CURRENT` | Referencia al documento actual en la etapa (por defecto igual a `$$ROOT`) |
| `$$this` | Referencia al elemento actual en `$map`, `$filter`, `$reduce` |
| `$$REMOVE` | Elimina el campo del documento (usado en `$project` o `$addFields`) |
| `$$NOW` | Fecha y hora actual del sistema |
| `$$CLUSTER_TIME` | Fecha actual del cluster (consistente entre nodos) |

```javascript
// $$ROOT: preservar el documento original tras $group
db.ventas.aggregate([
    { $group: {
        _id: "$categoria",
        ventas: { $push: "$$ROOT" }
    }}
])

// $$REMOVE: eliminar campos condicionalmente
db.usuarios.aggregate([
    { $project: {
        nombre: 1,
        email: 1,
        telefono: {
            $cond: {
                if: { $eq: ["$tipo", "premium"] },
                then: "$telefono",
                else: "$$REMOVE"
            }
        }
    }}
])
```

## Ejemplos completos

### Ventas por categoria y mes

```javascript
db.ventas.aggregate([
    { $match: {
        fecha: {
            $gte: ISODate("2025-01-01"),
            $lt: ISODate("2026-01-01")
        }
    }},
    { $group: {
        _id: {
            categoria: "$categoria",
            anyo: { $year: "$fecha" },
            mes: { $month: "$fecha" }
        },
        total_ventas: { $sum: "$importe" },
        num_transacciones: { $sum: 1 },
        promedio: { $avg: "$importe" }
    }},
    { $sort: { "_id.anyo": 1, "_id.mes": 1, total_ventas: -1 } },
    { $project: {
        _id: 0,
        categoria: "$_id.categoria",
        anyo: "$_id.anyo",
        mes: "$_id.mes",
        total_ventas: { $round: ["$total_ventas", 2] },
        num_transacciones: 1,
        promedio: { $round: ["$promedio", 2] }
    }}
])
```

### Top 10 productos mas vendidos

```javascript
db.ventas.aggregate([
    { $match: { fecha: { $gte: ISODate("2025-01-01") } } },
    { $group: {
        _id: "$producto",
        unidades_vendidas: { $sum: "$cantidad" },
        ingreso_total: { $sum: { $multiply: ["$cantidad", "$precio_unitario"] } }
    }},
    { $sort: { unidades_vendidas: -1 } },
    { $limit: 10 },
    { $lookup: {
        from: "productos",
        localField: "_id",
        foreignField: "_id",
        as: "info_producto"
    }},
    { $unwind: { path: "$info_producto", preserveNullAndEmptyArrays: true } },
    { $project: {
        producto: "$info_producto.nombre",
        unidades_vendidas: 1,
        ingreso_total: 1
    }}
])
```

### Estadisticas de usuarios activos

```javascript
db.usuarios.aggregate([
    { $match: { activo: true, ultimo_acceso: { $gte: ISODate("2025-06-01") } } },
    { $group: {
        _id: "$pais",
        total_usuarios: { $sum: 1 },
        edad_promedio: { $avg: "$edad" },
        edad_minima: { $min: "$edad" },
        edad_maxima: { $max: "$edad" },
        desviacion_edad: { $stdDevPop: "$edad" },
        planes: { $addToSet: "$plan" }
    }},
    { $sort: { total_usuarios: -1 } }
])
```

### Reporte mensual con multiples metricas

```javascript
db.ventas.aggregate([
    { $match: {
        fecha: { $gte: ISODate("2025-01-01"), $lt: ISODate("2026-01-01") }
    }},
    { $facet: {
        ventas_por_mes: [
            { $group: {
                _id: { mes: { $month: "$fecha" } },
                total: { $sum: "$importe" },
                transacciones: { $sum: 1 }
            }},
            { $sort: { "_id.mes": 1 } }
        ],
        top_categorias: [
            { $group: {
                _id: "$categoria",
                total: { $sum: "$importe" }
            }},
            { $sort: { total: -1 } },
            { $limit: 5 }
        ],
        resumen_general: [
            { $group: {
                _id: null,
                total_ventas: { $sum: "$importe" },
                promedio_venta: { $avg: "$importe" },
                total_transacciones: { $sum: 1 }
            }}
        ]
    }},
    { $project: {
        ventas_por_mes: 1,
        top_categorias: 1,
        resumen_general: { $arrayElemAt: ["$resumen_general", 0] }
    }}
])
```

La combinacion de `$facet` con `$project` permite extraer el unico objeto del array `resumen_general` para tener un campo plano en lugar de un array de un elemento.

