# Aggregation Framework

El aggregation framework de MongoDB es un motor de procesamiento de datos que permite transformar, filtrar, agrupar y analizar documentos dentro de un pipeline secuencial de etapas. A diferencia del comando `find`, que devuelve documentos tal cual se almacenan, el aggregation framework puede remodelar documentos, calcular agregaciones estadisticas, unir colecciones y generar resultados analiticos complejos. Esta disenado para operar de forma eficiente sobre grandes volumenes de datos directamente en el servidor de base de datos, evitando mover datos pesados a la capa de aplicacion.

## Pipeline de agregacion

El pipeline es una secuencia ordenada de etapas (stages) donde los documentos fluyen de una etapa a la siguiente. Cada etapa recibe un conjunto de documentos, los transforma segun su logica interna y pasa los documentos resultantes a la etapa siguiente. El resultado final es el conjunto de documentos que sale de la ultima etapa.

```
db.coleccion.aggregate([
    { $match:   { status: "activo" } },
    { $group:   { _id: "$categoria", total: { $sum: "$cantidad" } } },
    { $sort:    { total: -1 } },
    { $limit:   10 }
])
```

Cada elemento del array es un documento que describe una etapa. MongoDB procesa las etapas en el orden en que aparecen. El motor de agregacion puede paralelizar ciertas etapas cuando es posible, pero la semantica del pipeline es estrictamente secuencial desde la perspectiva del resultado.

> El pipeline se asemeja al concepto de tuberias (pipe) en Unix: cada etapa recibe datos, los procesa y los envía a la siguiente. Esta mentalidad compositiva permite construir analisis complejos combinando etapas simples y reutilizables.

## Etapas secuenciales

Cada etapa realiza una operacion especifica y su comportamiento esta determinado por el documento de especificacion de esa etapa. La salida de una etapa se convierte en la entrada de la siguiente. Un documento que pasa por una etapa puede:

- Pasar intacto a la siguiente etapa
- Ser filtrado y no continuar (etapas de filtrado como `$match`)
- Ser transformado: campos anyadidos, eliminados, renombrados o calculados (`$project`, `$addFields`)
- Ser descompuesto en multiples documentos (`$unwind`)
- Ser agrupado con otros documentos (`$group`)
- Ser escrito a una coleccion y terminar el pipeline (`$out`, `$merge`)

```
Documentos originales
    |
    v
[$match]    -> solo documentos que cumplen condicion
    |
    v
[$group]    -> documentos agrupados con acumuladores
    |
    v
[$project]  -> forma final de cada documento
    |
    v
[$sort]     -> orden ascendente/descendente
    |
    v
Resultado
```

## Optimizacion del pipeline

MongoDB aplica optimizaciones automaticas al pipeline antes de ejecutarlo. Sin embargo, el orden y la estructura que define el desarrollador influyen significativamente en el rendimiento. Las dos optimizaciones mas importantes que el desarrollador debe considerar son: colocar `$match` y `$project` lo antes posible.

### $match temprano

Si el pipeline comienza con un `$match`, MongoDB puede usar indices de la coleccion para reducir el conjunto de documentos antes de cualquier otra operacion. Esto es especialmente importante en pipelines con volumenes grandes de datos.

```javascript
// Ineficiente: se procesan todos los documentos antes de filtrar
db.ventas.aggregate([
    { $group: { _id: "$categoria", total: { $sum: "$importe" } } },
    { $match: { total: { $gte: 1000 } } }
])

// Eficiente: se filtra antes de agrupar
db.ventas.aggregate([
    { $match: { fecha: { $gte: ISODate("2025-01-01") } } },
    { $group: { _id: "$categoria", total: { $sum: "$importe" } } }
])
```

> Un `$match` en la primera etapa del pipeline se comporta como un `find()` y puede utilizar cualquier indice disponible en la coleccion. MongoDB puede incluso mover `$match` a la primera posicion como parte de sus optimizaciones automaticas cuando detecta que no altera el resultado.

### $project temprano

Reducir el tamano de los documentos lo antes posible con `$project` disminuye la cantidad de datos que las etapas posteriores deben procesar en memoria.

```javascript
// Eliminar campos innecesarios antes de $group
db.pedidos.aggregate([
    { $project: { cliente: 1, importe: 1, fecha: 1 } },
    { $group: { _id: "$cliente", total: { $sum: "$importe" } } }
])
```

### Otras optimizaciones automaticas

- **Coalescencia de `$sort` + `$limit`**: si hay un `$sort` seguido de `$limit`, MongoDB solo mantiene los `n` documentos top en memoria durante la ordenacion, evitando ordenar toda la coleccion.
- **Reordenacion de `$project`**: MongoDB puede mover `$project` antes de ciertas etapas si determina que es seguro hacerlo.
- **Uso de indices en `$sort`**: si el pipeline comienza con `$sort` y hay un indice que soporta el orden, MongoDB evita la ordenacion en memoria.

```javascript
// $sort + $limit se optimiza automaticamente: solo ordena los top 5
db.productos.aggregate([
    { $sort: { precio: -1 } },
    { $limit: 5 }
])
```

La optimizacion de pipelines no es solo responsabilidad del motor de MongoDB: el conocimiento del dominio de datos y los patrones de consulta permite disenar pipelines que aprovechen al maximo estas optimizaciones. En general, la regla es reducir el volumen de datos lo antes posible y usar proyecciones para eliminar campos innecesarios en etapas tempranas.

