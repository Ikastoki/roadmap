# Atlas Search

Atlas Search es un motor de busqueda full-text integrado directamente en MongoDB Atlas que utiliza Apache Lucene como backend de indexacion y consulta. A diferencia de las busquedas con el operador `$text` de MongoDB, que ofrece capacidades limitadas de tokenizacion y ranking, Atlas Search proporciona un analizador lexico completo, busqueda por relevancia, matches difusos (fuzzy), agregacion por facetas, resaltado de resultados y busqueda geoespacial combinada con texto. Todo esto sin necesidad de mantener un motor de busqueda separado como Elasticsearch o Solr.

La integracion con MongoDB es nativa: los indices de Atlas Search se definen sobre colecciones existentes y se sincronizan automaticamente con los cambios en los datos. Cuando un documento se inserta, actualiza o elimina, Atlas Search actualiza el indice Lucene correspondiente en tiempo real. Las consultas se ejecutan a traves del pipeline de agregacion de MongoDB, utilizando el operador `$search` como una etapa mas del pipeline. Esto permite combinar busquedas de texto con filtros, proyecciones, ordenaciones y agrupaciones del aggregation framework.

> Atlas Search no esta disponible en el tier gratuito M0. Requiere un cluster M10 o superior, y consume recursos adicionales de CPU y memoria en los nodos del replica set donde se ejecuta.

## Creacion de indices

Los indices de Atlas Search se definen mediante la interfaz web de Atlas o mediante la API. A diferencia de los indices de MongoDB, que se crean con `createIndex()`, los indices de Search se configuran en la pestana "Atlas Search" de la consola y se almacenan en la capa de metadatos de Atlas, no en la coleccion.

Cada indice de Search especifica:

- **Coleccion**: la coleccion MongoDB sobre la que se construye el indice.
- **Analizador**: el analizador lexico que procesa el texto. Atlas incluye analizadores para numerosos idiomas (`lucene.english`, `lucene.spanish`, `lucene.french`, etc.), ademas de analizadores genericos como `lucene.standard` y `lucene.keyword`.
- **Campos indexados**: la lista de campos que se indexan para busqueda, con su tipo (`string`, `number`, `date`, `boolean`, `objectId`, etc.) y configuracion de indexacion especifica (si se aplica stemming, si se almacena el termino original para resaltado, si se indexa para autocompletado, etc.).
- **Synonyms**: configuracion opcional de sinonimos para expandir las busquedas a terminos equivalentes.

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "titulo": {
        "type": "string",
        "analyzer": "lucene.spanish",
        "search": true,
        "store": true
      },
      "descripcion": {
        "type": "string",
        "analyzer": "lucene.spanish",
        "search": true,
        "store": true
      },
      "precio": {
        "type": "number"
      }
    }
  }
}
```

Una vez creado, el indice comienza el proceso de sincronizacion inicial. Para colecciones grandes, este proceso puede tardar minutos u horas, dependiendo del tamano de los datos y de los recursos del cluster. Los cambios posteriores se reflejan en el indice con una latencia de segundos.

## El operador $search

Las consultas a Atlas Search se realizan mediante el operador `$search` en el aggregation pipeline. La sintaxis basica combina el nombre del indice con uno o varios operadores de busqueda:

```javascript
db.productos.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "portatil gaming",
        path: ["titulo", "descripcion"],
        fuzzy: { maxEdits: 2 }
      }
    }
  },
  { $limit: 10 },
  { $project: { titulo: 1, precio: 1, score: { $meta: "searchScore" } } }
]);
```

La etapa `$search` debe ser la primera etapa del pipeline (o ir inmediatamente despues de un `$match` que utilice el indice de MongoDB), y el campo `score` esta disponible en los documentos de resultado para ordenar por relevancia.

### Operadores de busqueda

Atlas Search ofrece multiples operadores que se combinan para construir consultas complejas:

- **text**: busqueda de texto completo con analisis lexico. Soporta `query` (termino de busqueda), `path` (campos donde buscar), `fuzzy` (busqueda difusa), `score` (modificacion del score) y `synonyms` (expansion por sinonimos).
- **compound**: combina multiples operadores con clausulas `must`, `should`, `mustNot` y `filter`. Es el equivalente al bool query de Elasticsearch.
- **phrase**: busqueda por frase exacta, respetando el orden de las palabras. Soporta `slop` para permitir distancia entre terminos.
- **wildcard**: busqueda con patrones comodin (`?` para un caracter, `*` para varios).
- **regex**: busqueda con expresiones regulares sobre el indice.
- **range**: filtro por rango de valores numericos o de fecha.
- **geoShape** y **geoWithin**: busqueda geoespacial combinada con texto.
- **near**: busqueda por proximidad geografica.
- **facet**: recuento de resultados por categorias (facetas).
- **autocomplete**: autocompletado en tiempo real mientras el usuario escribe.
- **moreLikeThis**: documentos similares a un documento de ejemplo.
- **embeddedDocument**: busqueda en documentos embebidos.

La combinacion de operadores mediante `compound` es la forma mas potente de construir consultas:

```javascript
db.productos.aggregate([
  {
    $search: {
      index: "default",
      compound: {
        must: [
          { text: { query: "portatil", path: "titulo", fuzzy: {} } }
        ],
        filter: [
          { range: { path: "precio", gte: 500, lte: 2000 } }
        ],
        should: [
          { text: { query: "gaming", path: "descripcion" } }
        ]
      }
    }
  }
]);
```

## Busqueda difusa (fuzzy)

La busqueda difusa permite encontrar documentos aunque el termino de busqueda no coincida exactamente con el texto indexado. Utiliza la distancia de Levenshtein para medir la diferencia entre cadenas: un `maxEdits` de 1 permite un error de un caracter (insercion, eliminacion o sustitucion), mientras que `maxEdits` de 2 permite dos errores.

```javascript
$search: {
  index: "default",
  text: {
    query: "portatil",
    path: "titulo",
    fuzzy: {
      maxEdits: 2,
      prefixLength: 3,
      maxExpansions: 50
    }
  }
}
```

Los parametros adicionales de la busqueda difusa son:

- **prefixLength**: numero de caracteres iniciales que deben coincidir exactamente. Un valor de 3 evita falsos positivos en palabras cortas.
- **maxExpansions**: limite de variaciones de la palabra que se evaluan. Controla el rendimiento de la consulta; un valor muy alto puede degradar la latencia.

La busqueda difusa es especialmente util para corregir errores tipograficos del usuario sin necesidad de implementar un corrector ortografico externo. En combinacion con `compound.should`, los resultados exactos puntuan mas alto que los difusos, manteniendo la relevancia.

## Busqueda por facetas

Las facetas permiten agrupar los resultados de busqueda en categorias con recuento de documentos, similar a los filtros laterales que ofrecen los sitios de comercio electronico. Atlas Search implementa las facetas mediante el operador `facet` dentro de `$search`:

```javascript
db.productos.aggregate([
  {
    $search: {
      index: "default",
      facet: {
        operators: [
          { text: { query: "portatil", path: "titulo" } }
        ],
        facets: {
          categorias: {
            type: "string",
            path: "categoria",
            numBuckets: 10
          },
          rango_precios: {
            type: "number",
            path: "precio",
            boundaries: [0, 500, 1000, 2000, 5000],
            default: "other"
          },
          anyo_lanzamiento: {
            type: "date",
            path: "fecha_lanzamiento",
            boundaries: [
              ISODate("2020-01-01"),
              ISODate("2021-01-01"),
              ISODate("2022-01-01"),
              ISODate("2023-01-01"),
              ISODate("2024-01-01")
            ]
          }
        }
      }
    }
  },
  {
    $facet: {
      resultados: [{ $limit: 20 }],
      facetas: [{ $replaceWith: "$$SEARCH_META" }, { $limit: 1 }]
    }
  }
]);
```

La variable `$$SEARCH_META` contiene los recuentos de cada faceta, permitiendo devolver tanto los resultados como las facetas en una sola consulta. La aplicacion puede entonces mostrar los resultados filtrados y, en la barra lateral, las opciones de filtro con el numero de documentos que coinciden con cada una.

## Resaltado de resultados

Atlas Search puede devolver fragmentos del texto que coinciden con la busqueda, marcados con el contexto circundante. Esto se configura en la etapa `$search` con la opcion `highlight`:

```javascript
db.productos.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "portatil gaming",
        path: "descripcion",
        highlight: { path: "descripcion" }
      }
    }
  },
  {
    $project: {
      titulo: 1,
      descripcion: 1,
      highlights: { $meta: "searchHighlights" }
    }
  }
]);
```

El resultado incluye un array de fragmentos con las posiciones de los terminos coincidentes, que la aplicacion puede procesar para envolver en etiquetas HTML como `<mark>` o `<strong>`.

## Integracion con el aggregation pipeline

La potencia real de Atlas Search emerge cuando se combina con el resto del aggregation pipeline. Dado que `$search` es una etapa mas del pipeline, puede ir seguida de `$match`, `$group`, `$lookup`, `$sort` y cualquier otra etapa de agregacion.

Un patron comun es realizar primero la busqueda full-text con `$search`, filtrar los resultados con `$match` utilizando campos indexados en MongoDB, enriquecer los datos con `$lookup` desde otras colecciones y finalmente agrupar o proyectar los resultados:

```javascript
db.pedidos.aggregate([
  {
    $search: {
      index: "busqueda_pedidos",
      text: { query: "factura atrasada", path: "notas" }
    }
  },
  { $match: { estado: "pendiente", total: { $gt: 100 } } },
  {
    $lookup: {
      from: "clientes",
      localField: "cliente_id",
      foreignField: "_id",
      as: "cliente"
    }
  },
  { $unwind: "$cliente" },
  {
    $group: {
      _id: "$cliente.region",
      total_pendiente: { $sum: "$total" },
      pedidos: { $push: { id: "$_id", total: "$total" } }
    }
  },
  { $sort: { total_pendiente: -1 } }
]);
```

Esta capacidad de combinar busqueda textual con analitica directamente en la base de datos elimina la necesidad de sincronizar datos con un motor de busqueda externo, simplificando la arquitectura y reduciendo la latencia de los datos entre sistemas.

