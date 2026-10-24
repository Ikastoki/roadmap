# Patrones de Diseno en MongoDB

El exito de un sistema basado en MongoDB no depende unicamente de conocer su modelo de datos o su lenguaje de consultas, sino de aplicar los patrones de diseno adecuados a cada problema. Estos patrones emergen de la practica real en produccion y responden a desafios recurrentes: esquemas que evolucionan, datos temporales masivos, documentos que crecen sin control, jerarquias complejas o consultas que cruzan multiples colecciones. Lejos de ser recetas abstractas, constituyen decisiones arquitectonicas que determinan el rendimiento, la escalabilidad y la mantenibilidad del sistema.

## Schema Versioning

Cuando una aplicacion en produccion necesita evolucionar su esquema de datos, el enfoque tradicional de migraciones SQL (ALTER TABLE) resulta disruptivo en MongoDB, especialmente en sistemas con alta disponibilidad o sin ventanas de mantenimiento. El patron de versionado de esquema aborda este problema incluyendo un campo `schema_version` en cada documento que indica la version del esquema que ese documento sigue.

La implementacion tipica consiste en anadir un campo numerico:

```json
{
  "schema_version": 2,
  "nombre": "Juan",
  "email": "juan@email.com",
  "telefono": "+34 600 000 000"
}
```

Mientras la version 1 almacenaba nombre y email unicamente, la version 2 agrega telefono. La logica de la aplicacion debe manejar ambas versiones durante la transicion. Las estrategias de migracion pueden ser:

- **Lazy migration**: los documentos se actualizan al ser leidos, sin necesidad de un proceso batch.
- **Background migration**: un worker recorre documentos antiguos y los actualiza progresivamente.
- **Mixed read**: la capa de aplicacion interpreta cada version y normaliza la salida.

Este patron evita downtime y permite despliegues graduales, pero obliga a mantener compatibilidad hacia atras durante el periodo de transicion. Es especialmente util en sistemas SaaS donde no se puede controlar el momento de actualizacion de todos los clientes.

## Bucket Pattern

El patron Bucket agrupa documentos relacionados en un unico contenedor basado en un intervalo temporal o un limite de tamano. Es la alternativa natural al almacenamiento de documentos individuales por evento en escenarios time-series.

En lugar de crear un documento por cada lectura de sensor:

```json
{
  "sensor_id": "sensor-01",
  "timestamp": ISODate("2026-07-28T10:00:00Z"),
  "valor": 23.5
}
{
  "sensor_id": "sensor-01",
  "timestamp": ISODate("2026-07-28T10:01:00Z"),
  "valor": 23.7
}
```

Se agrupan en un bucket por hora:

```json
{
  "sensor_id": "sensor-01",
  "bucket_hora": ISODate("2026-07-28T10:00:00Z"),
  "lecturas": [
    { "t": 0, "v": 23.5 },
    { "t": 60, "v": 23.7 }
  ],
  "total_lecturas": 60
}
```

Las ventajas son multiples: reduccion del numero de documentos en varios ordenes de magnitud, mejor uso del espacio en disco (los documentos comparten el campo sensor_id y el timestamp del bucket), indices mas pequenos y lecturas mas eficientes al recuperar bloques completos de datos con una sola consulta. La contrapartida es la complejidad adicional en las operaciones de escritura, que requieren actualizar arrays en lugar de hacer inserts simples.

## Subset Pattern

Cuando los documentos contienen campos que se acceden con frecuencias muy dispares, el patron Subset propone dividir la informacion en dos colecciones: una con los datos de acceso frecuente y otra con los datos de acceso esporadico.

Un caso tipico son los catalogos de productos. La consulta de listados solo necesita nombre, precio e imagen, mientras que las descripciones extensas, especificaciones tecnicas y manuales solo se consultan en la pagina de detalle:

**Coleccion productos_resumen:**
```json
{
  "_id": ObjectId("..."),
  "nombre": "Laptop X",
  "precio": 1200,
  "imagen": "url-imagen.jpg",
  "stock": 15
}
```

**Coleccion productos_detalle:**
```json
{
  "_id": ObjectId("..."),
  "producto_id": ObjectId("..."),
  "descripcion": "Laptop con procesador...",
  "especificaciones": { "ram": "32GB", "disco": "1TB SSD" },
  "manual_url": "url-manual.pdf",
  "opiniones": [ ... ]
}
```

Este patron reduce el tamano de los documentos en memoria (working set) y acelera las consultas mas frecuentes. Es especialmente util cuando la relacion de acceso entre datos calientes y frios supera 10:1.

## Outlier Pattern

Los documentos outlier (atipicos) son aquellos que exceden el tamano tipico de la mayoria de documentos en una coleccion, ya sea por arrays que crecen sin cota, por campos de texto extensos o por relaciones que escalan desproporcionadamente. El patron Outlier consiste en detectar estos casos y mover los datos exceptionales a una coleccion separada, manteniendo un puntero o referencia.

Por ejemplo, en una red social, la mayoria de usuarios tienen entre 0 y 100 seguidores, pero un usuario celebridad puede tener millones. En lugar de almacenar el array de seguidores en el documento del usuario (que superaria el limite de 16 MB), se separa:

```json
// Coleccion usuarios (documento normal)
{
  "_id": ObjectId("..."),
  "nombre": "Usuario Normal",
  "num_seguidores": 45,
  "seguidores_id": "normal"
}

// Coleccion usuarios_outlier
{
  "_id": ObjectId("..."),
  "usuario_id": ObjectId("..."),
  "seguidores": [ ObjectId("..."), ObjectId("..."), ... ]
}
```

La logica de aplicacion consulta primero la coleccion principal y, si detecta el marcador outlier, recupera los datos adicionales de la coleccion secundaria. Este patron mantiene el rendimiento predecible y evita que casos extremos degraden la experiencia del resto de usuarios.

## Computed Pattern

El patron Computed precalcula y almacena valores derivados que de otro modo requeririan costosas operaciones de agregacion en tiempo real. Es aplicable siempre que existan calculos repetitivos sobre los mismos datos: totales, promedios, conteos, rankings o cualquier agregacion que no necesite reflejar el estado exacto en milisegundos.

La estrategia consiste en ejecutar el calculo en un proceso background (tipicamente un worker o una funcion periodica) y almacenar el resultado en un campo del documento o en una coleccion separada.

```json
// Pedido con total precalculado
{
  "_id": ObjectId("..."),
  "cliente_id": ObjectId("..."),
  "articulos": [
    { "producto": "A", "precio": 10, "cantidad": 2 },
    { "producto": "B", "precio": 15, "cantidad": 1 }
  ],
  "total": 35,
  "total_impuestos": 7.35
}
```

Si se necesitara calcular `total` en cada consulta mediante el aggregation framework, el coste creceria linealmente con el volumen de pedidos. Almacenarlo precalculado reduce la latencia de lectura a una unica busqueda por _id. Las actualizaciones pueden hacerse mediante atomic increments, actualizaciones periodicas con correccion o mediante change streams que disparan el recalculado.

## Extended Reference Pattern

Este patron replica campos de uso frecuente de una coleccion referenciada directamente en el documento que los necesita, con el objetivo de eliminar la necesidad de usar `$lookup` (equivalente a un JOIN) en las consultas.

En lugar de tener un pedido que referencia al cliente y requerir un `$lookup` cada vez que se muestra el nombre del cliente:

```json
// Coleccion pedidos sin extended reference
{
  "_id": ObjectId("..."),
  "cliente_id": ObjectId("..."),
  "fecha": ISODate("2026-07-28"),
  "total": 200
}
```

Se almacena una copia de los campos mas consultados:

```json
{
  "_id": ObjectId("..."),
  "cliente_id": ObjectId("..."),
  "cliente_nombre": "Juan Perez",
  "cliente_email": "juan@email.com",
  "fecha": ISODate("2026-07-28"),
  "total": 200
}
```

`$lookup` es costoso porque obliga a MongoDB a ejecutar una segunda consulta o incluso un merge en memoria. Para colecciones con millones de documentos, su uso indiscriminado degrada el rendimiento gravemente. Extended Reference reduce las lecturas a una unica consulta a costa de duplicar datos y requerir actualizaciones sincronizadas cuando los campos replicados cambian. Es un compromiso aceptable cuando la frecuencia de lectura supera ampliamente a la de escritura.

## Attribute Pattern

El patron Attribute maneja escenarios donde los atributos de una entidad son variables, impredecibles o numerosos. En lugar de crear un campo por cada posible atributo (lo que llevaria a esquemas rigidos o a muchos campos nulos), se modela como un array de pares clave-valor.

Un ejemplo clasico son los productos de un e-commerce. Un portatil tiene atributos como RAM, procesador y disco, mientras que una camiseta tiene talla, color y tejido. Con el patron Attribute:

```json
{
  "_id": ObjectId("..."),
  "nombre": "Laptop X",
  "categoria": "electronica",
  "atributos": [
    { "k": "ram", "v": "32GB" },
    { "k": "procesador", "v": "Intel i7" },
    { "k": "disco", "v": "1TB SSD" }
  ]
}
```

Para filtrar productos por un atributo especifico, se usa un indice multikey sobre `atributos.k` y `atributos.v`, permitiendo consultas eficientes como:

```javascript
db.productos.find({
  "atributos": { $elemMatch: { k: "ram", v: "32GB" } }
})
```

Este patron es fundamental en sistemas de catalogos, configuraciones dinamicas, metadatos de archivos, datos de formularios dinamicos y cualquier dominio donde el esquema de los datos no pueda fijarse en la capa de aplicacion.

## Tree Patterns

MongoDB carece de joins recursivos nativos, por lo que el modelado de jerarquias (arboles, categorias, organigramas, comentarios anidados) requiere patrones especificos. El mas recomendado en la mayoria de los casos es **Materialized Path**, que almacena la ruta completa desde la raiz hasta cada nodo.

```json
{
  "_id": ObjectId("..."),
  "nombre": "Electronica",
  "path": ",electronica,",
  "nivel": 0
}
{
  "_id": ObjectId("..."),
  "nombre": "Portatiles",
  "path": ",electronica,portatiles,",
  "nivel": 1
}
{
  "_id": ObjectId("..."),
  "nombre": "Ultrabooks",
  "path": ",electronica,portatiles,ultrabooks,",
  "nivel": 2
}
```

Para obtener todos los descendientes de una categoria basta con una busqueda por expresion regular sobre `path`:

```javascript
db.categorias.find({ path: /,electronica,/ })
```

`$graphLookup`, introducido en MongoDB 3.4, permite recorrer grafos y arboles almacenados como referencias padre-hijo (Parent Reference o Adjacency List). Es util para arboles profundos donde el materialized path resultaria en cadenas muy largas, aunque tiene limitaciones de profundidad (maximo 100 niveles por defecto) y rendimiento inferior al materialized path para lecturas.

La eleccion entre Materialized Path y `$graphLookup` depende del caso: el primero es optimo para lecturas frecuentes de subarboles completos; el segundo, para arboles que se actualizan constantemente y requieren integridad referencial.

