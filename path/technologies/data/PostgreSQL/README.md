# PostgreSQL

## Índice

- [Introducción a las bases de datos](#introducción-a-las-bases-de-datos)
- [Introducción a PostgreSQL](#introducción-a-postgresql)
- [Instalación y configuración](#instalación-y-configuración)
- [SQL: DDL y tipos de datos](#sql-ddl-y-tipos-de-datos)
- [SQL: DML y consultas](#sql-dml-y-consultas)
- [Funciones y expresiones](#funciones-y-expresiones)
- [Índices](#índices)
- [Vistas](#vistas)
- [Funciones y procedimientos](#funciones-y-procedimientos)
- [Triggers](#triggers)
- [Transacciones y concurrencia](#transacciones-y-concurrencia)
- [Full-text search, JSON y arrays](#full-text-search-json-y-arrays)
- [Rendimiento y particionamiento](#rendimiento-y-particionamiento)
- [Backup y replicación](#backup-y-replicación)
- [Seguridad](#seguridad)
- [Extensiones y monitorización](#extensiones-y-monitorización)
- [Buenas prácticas](#buenas-prácticas)

> [!WARNING]
> Los apuntes no tienen acentos ni ñ porque, al probar una extensión nueva con IA, los ha borrado. Cuando termine de subir todo, lo solucionaré.

## Introducción a las bases de datos

- [Qué es una base de datos](01-introduccion_bd/01-que_es_bd.md)
- [Modelos de datos](01-introduccion_bd/02-modelos_de_datos.md)
- [SGBD y PostgreSQL](01-introduccion_bd/03-sgbd_y_postgresql.md)

## Introducción a PostgreSQL

- [Qué es PostgreSQL](02-introduccion_postgres/01-que_es_postgresql.md)
- [Arquitectura de PostgreSQL](02-introduccion_postgres/02-arquitectura.md)

## Instalación y configuración

- [Instalación](03-instalacion_config/01-instalacion.md)
- [Configuración](03-instalacion_config/02-configuracion.md)
- [Herramientas](03-instalacion_config/03-herramientas.md)

## SQL: DDL y tipos de datos

- [Introducción a SQL](04-sql_ddl_tipos/01-introduccion_sql.md)
- [DDL: tablas](04-sql_ddl_tipos/02-ddl_tablas.md)
- [Tipos numéricos y de texto](04-sql_ddl_tipos/03-tipos_numericos_texto.md)
- [Tipos de fecha, red, geo y compuestos](04-sql_ddl_tipos/04-tipos_fecha_geo_compuestos.md)
- [Restricciones y relaciones](04-sql_ddl_tipos/05-restricciones_y_relaciones.md)

## SQL: DML y consultas

- [INSERT, UPDATE y DELETE](05-sql_dml_select/01-dml_insert_update_delete.md)
- [TCL y DCL](05-sql_dml_select/02-tcl_dcl.md)
- [SELECT básico](05-sql_dml_select/03-select_basico.md)
- [Agregación y funciones ventana](05-sql_dml_select/04-agregacion_window_functions.md)
- [JOINs y subconsultas](05-sql_dml_select/05-joins_y_subconsultas.md)

## Funciones y expresiones

- [Funciones de texto, numéricas y de fecha](06-funciones_expresiones/01-funciones_texto_numericas_fecha.md)
- [Expresiones condicionales](06-funciones_expresiones/02-expresiones_condicionales.md)

## Índices

- [Concepto y tipos de índices](07-indices/01-concepto_y_tipos.md)
- [Índices parciales, de expresión, multicolumna y únicos](07-indices/02-indices_parciales_expresion.md)
- [Mantenimiento de índices](07-indices/03-mantenimiento.md)

## Vistas

- [Vistas](08-vistas/01-vistas.md)
- [Vistas materializadas](08-vistas/02-vistas_materializadas.md)

## Funciones y procedimientos

- [Introducción a PL/pgSQL](09-funciones_procedimientos/01-introduccion_plpgsql.md)
- [Funciones](09-funciones_procedimientos/02-funciones.md)
- [Procedimientos](09-funciones_procedimientos/03-procedimientos.md)
- [Control de flujo](09-funciones_procedimientos/04-control_flujo.md)
- [Cursores y manejo de errores](09-funciones_procedimientos/05-cursores_y_errores.md)

## Triggers

- [Concepto de triggers](10-triggers/01-concepto_triggers.md)
- [Ejemplos y gestión](10-triggers/02-ejemplos_gestion.md)

## Transacciones y concurrencia

- [Transacciones](11-transacciones_concurrencia/01-transacciones.md)
- [Niveles de aislamiento y bloqueos](11-transacciones_concurrencia/02-niveles_aislamiento_bloqueos.md)
- [MVCC](11-transacciones_concurrencia/03-mvcc.md)

## Full-text search, JSON y arrays

- [Full-text search](12-fulltext_json_arrays/01-fulltext_search.md)
- [JSON y JSONB](12-fulltext_json_arrays/02-json.md)
- [JSON avanzado](12-fulltext_json_arrays/03-json_avanzado.md)
- [Arrays](12-fulltext_json_arrays/04-arrays.md)

## Rendimiento y particionamiento

- [EXPLAIN y estadísticas](13-rendimiento_particionamiento/01-explain_y_estadisticas.md)
- [Configuración de rendimiento](13-rendimiento_particionamiento/02-configuracion_rendimiento.md)
- [VACUUM](13-rendimiento_particionamiento/03-vacuum.md)
- [Particionamiento](13-rendimiento_particionamiento/04-particionamiento.md)

## Backup y replicación

- [Backup](14-backup_replicacion/01-backup.md)
- [Restauración y Point-in-Time Recovery](14-backup_replicacion/02-restauracion_pitr.md)
- [Streaming replication](14-backup_replicacion/03-streaming_replication.md)
- [Logical replication y alta disponibilidad](14-backup_replicacion/04-logical_replication_ha.md)

## Seguridad

- [Autenticación y autorización](15-seguridad/01-autenticacion_autorizacion.md)
- [Row-Level Security](15-seguridad/02-row_level_security.md)
- [Cifrado y auditoría](15-seguridad/03-cifrado_auditoria.md)

## Extensiones y monitorización

- [Extensiones](16-extensiones_monitoreo/01-extensiones.md)
- [Foreign Data Wrappers](16-extensiones_monitoreo/02-foreign_data_wrappers.md)
- [Monitorización y administración](16-extensiones_monitoreo/03-monitoreo_administracion.md)

## Buenas prácticas

- [Convenciones de nombrado](17-buenas_practicas/01-convenciones_naming.md)
- [Diseño de esquemas](17-buenas_practicas/02-diseno_esquemas.md)
- [Rendimiento](17-buenas_practicas/03-rendimiento.md)
- [Seguridad y mantenimiento](17-buenas_practicas/04-seguridad_mantenimiento.md)
