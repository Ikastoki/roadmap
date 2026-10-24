# Comparativa SQL vs NoSQL

La decision entre una base de datos relacional y una NoSQL no es tecnologica sino arquitectonica: responde a los requisitos de escalabilidad, consistencia, flexibilidad y modelo de datos que impone el dominio del problema. No existe una opcion universalmente superior; cada paradigma optimiza para un conjunto diferente de compensaciones (trade-offs).

## Diferencias fundamentales

### Esquema: fijo vs dinamico

**SQL** exige un esquema definido de antemano (schema-on-write). Cada columna tiene un nombre, tipo y restricciones que se validan en el momento de la escritura. Alterar el esquema requiere sentencias DDL (ALTER TABLE) que pueden bloquear la tabla y exigir migraciones.

**NoSQL** aplica schema-on-read: el esquema se infiere en el momento de la lectura. Cada documento puede tener campos distintos sin que la base de datos imponga validacion. Anyadir un nuevo campo solo requiere empezar a escribirlo en los nuevos documentos; los existentes simplemente devuelven null para ese campo si se consulta.

```
SQL:         INSERT INTO usuarios (nombre, email) VALUES ('Ana', 'ana@x.com')
             -- Error si falta columna obligatoria

NoSQL:       db.usuarios.insertOne({ nombre: "Ana", email: "ana@x.com" })
             db.usuarios.insertOne({ nombre: "Luis", telefono: "+34..." })
             -- Cada documento puede tener estructura diferente
```

> El esquema fijo de SQL es una ventaja cuando la integridad estructural es critica. El esquema dinamico de NoSQL acelera el desarrollo cuando los requisitos cambian con frecuencia.

### Joins vs embeddings

**SQL** normaliza los datos en tablas separadas y los relaciona mediante joins en el momento de la consulta. Esto evita redundancia pero introduce complejidad y coste en las lecturas, especialmente al escalar.

**NoSQL** denormaliza los datos incrustando (embedding) la informacion relacionada dentro del mismo documento. Una orden de compra contiene las lineas de detalle como un array anidado. Esto elimina los joins y acelera las lecturas, pero introduce redundancia y complica las actualizaciones.

```javascript
// Embedding en MongoDB - todo en un documento
{
  "_id": "ORD-001",
  "cliente": "Ana Garcia",
  "fecha": "2026-07-28",
  "lineas": [
    { "producto": "Laptop", "cantidad": 1, "precio": 1200 },
    { "producto": "Mouse",  "cantidad": 2, "precio": 25 }
  ],
  "total": 1250
}
```

```sql
-- Normalizacion en SQL
SELECT o.id, o.fecha, l.producto, l.cantidad
FROM ordenes o
JOIN lineas_orden l ON o.id = l.orden_id;
```

La decision entre embedding y referencia depende del patron de acceso: si los datos relacionados se leen siempre juntos, el embedding es eficiente. Si se accede por separado o las relaciones son muchos-a-muchos, la referencia (o una tabla intermedia en SQL) es mejor.

### ACID vs BASE

SQL garantiza ACID, lo que asegura que las transacciones son atomicas, consistentes, aisladas y duraderas. Esto es indispensable en dominios donde la integridad de los datos no puede comprometerse: contabilidad, reservas, sistemas financieros.

NoSQL adopta BASE, priorizando disponibilidad y rendimiento sobre consistencia inmediata. En un sistema BASE, una lectura puede devolver datos desactualizados durante un periodo breve hasta que la consistencia eventual se alcanza.

> En MongoDB, a partir de la version 4.0, es posible usar transacciones ACID multi-documento, lo que desdibuja la linea entre SQL y NoSQL. Sin embargo, el uso de transacciones en MongoDB tiene un coste de rendimiento y debe aplicarse solo cuando es estrictamente necesario.

### Escalabilidad: vertical vs horizontal

**SQL** escala verticalmente: se anade mas potencia al servidor existente (mas CPU, RAM, discos SSD). Esto tiene un limite fisico y economico. El clustering en SQL (sharding) es posible pero complejo de configurar y mantener, y muchas bases de datos relacionales no lo soportan de forma nativa.

**NoSQL** escala horizontalmente: se anaden mas servidores (nodos) al cluster y los datos se distribuyen automaticamente mediante sharding. Esto permite escalar de forma practicamente ilimitada usando hardware comodo (commodity hardware).

| Aspecto | SQL | NoSQL |
|---|---|---|
| Escalabilidad | Vertical (mas hardware) | Horizontal (mas nodos) |
| Esquema | Fijo (schema-on-write) | Dinamico (schema-on-read) |
| Relaciones | Joins, normalizacion | Embeddings, referencias |
| Transacciones | ACID | BASE (o ACID limitado) |
| Consistencia | Fuerte | Eventual (configurable) |
| Lenguaje | SQL estandar | Propietario (cada sistema) |
| Modelo | Tablas, filas, columnas | Documentos, grafos, clave-valor, columnas |
| Madurez | Decadas de desarrollo | Menor, evolucion rapida |

## Casos de uso tipicos

### SQL es la opcion adecuada para

- **Sistemas financieros y contables**: la consistencia fuerte y las transacciones ACID no son negociables. Una transferencia bancaria debe deducir de una cuenta y acreditar en otra atomicamente.
- **Aplicaciones con relaciones complejas**: ERP, CRM, sistemas de inventario donde las entidades se relacionan mediante claves foraneas con reglas de integridad referencial.
- **Reporting y BI**: las herramientas de business intelligence asumen un esquema relacional y SQL como lenguaje de consulta.
- **Datos estructurados y estables**: cuando el modelo de datos apenas cambia y se conoce desde el inicio del proyecto.

### NoSQL es la opcion adecuada para

- **Catalogo de productos e-commerce**: cada producto puede tener atributos diferentes (schema-less), y el catalogo debe escalar horizontalmente para soportar picos de trafico.
- **Gestion de sesiones y caching**: Redis o Memcached ofrecen latencia de microsegundos para almacenar sesiones de usuario.
- **Redes sociales y grafos de relaciones**: Neo4j permite consultar "amigos de amigos" con eficiencia constante, algo que en SQL requeriria joins recursivos costosos.
- **IoT y series temporales**: InfluxDB o TimescaleDB ingieren millones de puntos de datos por segundo con compresion y retencion automatica.
- **Big Data y analitica**: Cassandra y HBase estan disenadas para almacenar petabytes de datos en cientos de nodos.

> No es raro encontrar arquitecturas poliglotas donde conviven PostgreSQL (datos transaccionales), MongoDB (catalogo de productos), Redis (cache de sesiones) y Elasticsearch (busqueda de texto completo). Cada base de datos hace lo que mejor sabe hacer.

