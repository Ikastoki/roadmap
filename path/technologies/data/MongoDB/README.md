# MongoDB

## Índice

- [Introducción a NoSQL](#introducción-a-nosql)
- [Introducción a MongoDB](#introducción-a-mongodb)
- [Instalación y configuración](#instalación-y-configuración)
- [Modelo de datos](#modelo-de-datos)
- [Operaciones CRUD](#operaciones-crud)
- [Aggregation Framework](#aggregation-framework)
- [Índices](#índices)
- [Replicación](#replicación)
- [Sharding](#sharding)
- [Seguridad](#seguridad)
- [Transacciones](#transacciones)
- [Change Streams](#change-streams)
- [Rendimiento](#rendimiento)
- [Backup y restauración](#backup-y-restauración)
- [Drivers y herramientas](#drivers-y-herramientas)
- [MongoDB Atlas](#mongodb-atlas)
- [Patrones y casos de uso](#patrones-y-casos-de-uso)
- [Buenas prácticas](#buenas-prácticas)

> [!WARNING]
> Los apuntes no tienen acentos ni ñ porque, al probar una extensión nueva con IA, los ha borrado. Cuando termine de subir todo, lo solucionaré.

## Introducción a NoSQL

- [Qué es NoSQL](01-introduccion_nosql/01-que_es_nosql.md)
- [Tipos de bases de datos NoSQL](01-introduccion_nosql/02-tipos_de_nosql.md)
- [Comparativa SQL vs NoSQL](01-introduccion_nosql/03-comparativa_sql_vs_nosql.md)

## Introducción a MongoDB

- [Qué es MongoDB](02-introduccion_mongodb/01-que_es_mongodb.md)
- [Arquitectura](02-introduccion_mongodb/02-arquitectura.md)
- [WiredTiger](02-introduccion_mongodb/03-wiredtiger.md)

## Instalación y configuración

- [Instalación](03-instalacion_configuracion/01-instalacion.md)
- [Configuración](03-instalacion_configuracion/02-configuracion.md)
- [mongosh](03-instalacion_configuracion/03-mongosh.md)
- [Herramientas de administración](03-instalacion_configuracion/04-herramientas_admin.md)

## Modelo de datos

- [BSON y ObjectId](04-modelo_datos/01-bson_y_objectid.md)
- [Esquema flexible](04-modelo_datos/02-esquema_flexible.md)
- [Patrones de modelado](04-modelo_datos/03-patrones_modelado.md)
- [Relaciones entre documentos](04-modelo_datos/04-relaciones.md)

## Operaciones CRUD

- [Insertar documentos](05-crud/01-insert.md)
- [Consultar documentos](05-crud/02-find.md)
- [Actualizar documentos](05-crud/03-update.md)
- [Eliminar documentos](05-crud/04-delete.md)
- [Operaciones bulk](05-crud/05-bulk_operations.md)

## Aggregation Framework

- [Introducción al pipeline](06-aggregation_framework/01-introduccion_pipeline.md)
- [Etapas básicas](06-aggregation_framework/02-etapas_basicas.md)
- [Etapas avanzadas](06-aggregation_framework/03-etapas_avanzadas.md)
- [Operadores y ejemplos](06-aggregation_framework/04-operadores_y_ejemplos.md)

## Índices

- [Concepto de índices](07-indices/01-concepto.md)
- [Tipos de índices](07-indices/02-tipos_de_indices.md)
- [Gestión de índices](07-indices/03-gestion.md)
- [Índices compuestos](07-indices/04-indices_compuestos.md)
- [Mantenimiento de índices](07-indices/05-mantenimiento.md)

## Replicación

- [Concepto de Replica Set](08-replicacion/01-concepto_replica_set.md)
- [Configuración de un Replica Set](08-replicacion/02-configuracion.md)
- [Read Preferences y Write/Read Concerns](08-replicacion/03-read_write_concerns.md)

## Sharding

- [Concepto de sharding](09-sharding/01-concepto_sharding.md)
- [Configuración de un clúster sharded](09-sharding/02-configuracion.md)
- [Elección de la shard key](09-sharding/03-shard_key.md)
- [Gestión del clúster sharded](09-sharding/04-gestion.md)

## Seguridad

- [Autenticación](10-seguridad/01-autenticacion.md)
- [Autorización y roles](10-seguridad/02-autorizacion_roles.md)
- [Cifrado](10-seguridad/03-cifrado.md)
- [Seguridad de red y auditoría](10-seguridad/04-seguridad_red_auditoria.md)

## Transacciones

- [Transacciones](11-transactions/01-transacciones.md)
- [Cuándo usar transacciones](11-transactions/02-cuando_usar.md)

## Change Streams

- [Concepto de Change Streams](12-change_streams/01-concepto_change_streams.md)
- [Casos de uso](12-change_streams/02-casos_de_uso.md)

## Rendimiento

- [Análisis de rendimiento](13-performance/01-analisis_rendimiento.md)
- [Profiling](13-performance/02-profiling.md)
- [Optimización](13-performance/03-optimizacion.md)

## Backup y restauración

- [Métodos de backup](14-backup/01-metodos_backup.md)
- [Restauración de backups](14-backup/02-restauracion.md)
- [Estrategias de backup](14-backup/03-estrategias.md)

## Drivers y herramientas

- [Drivers oficiales](15-drivers_herramientas/01-drivers_oficiales.md)
- [Herramientas de administración](15-drivers_herramientas/02-herramientas_admin.md)
- [Integraciones](15-drivers_herramientas/03-integraciones.md)

## MongoDB Atlas

- [Qué es MongoDB Atlas](16-atlas/01-que_es_atlas.md)
- [Configuración de Atlas](16-atlas/02-configuracion_atlas.md)
- [Atlas Search](16-atlas/03-atlas_search.md)
- [Atlas avanzado](16-atlas/04-atlas_avanzado.md)

## Patrones y casos de uso

- [Patrones de diseño](17-patrones_casos_uso/01-patrones_de_diseno.md)
- [Casos de uso reales](17-patrones_casos_uso/02-casos_de_uso_reales.md)
- [Ejemplos prácticos](17-patrones_casos_uso/03-ejemplos_practicos.md)

## Buenas prácticas

- [Diseño de esquemas](18-buenas_practicas/01-diseno_esquemas.md)
- [Rendimiento](18-buenas_practicas/02-rendimiento.md)
- [Seguridad](18-buenas_practicas/03-seguridad.md)
- [Operaciones](18-buenas_practicas/04-operaciones.md)
- [Desarrollo](18-buenas_practicas/05-desarrollo.md)
