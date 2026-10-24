# SQL: lenguaje de consultas estructurado

El SQL (_Structured Query Language_) es el lenguaje universal para la comunicacion con bases de datos relacionales. Su origen se remonta a principios de los anos 70, cuando Donald D. Chamberlin y Raymond F. Boyce, investigadores de IBM, desarrollaron SEQUEL (_Structured English Query Language_) como parte del proyecto System R, el primer prototipo funcional de un SGBD relacional. Aunque IBM registro SEQUEL como marca, por conflictos legales el nombre se acorto a SQL, y asi llego hasta nuestros dias como el estandar indiscutible para el manejo de datos relacionales. Lejos de ser un lenguaje estatico, SQL ha evolucionado durante mas de cinco decadas mediante la accion de los comites ANSI e ISO, incorporando progresivamente desde funciones de ventana hasta soporte para JSON, y definiendo con cada revision el rumbo tecnologico de toda una industria.

## Historia y estandarizacion

La primera version comercialmente relevante llego en 1979 con Oracle (entonces Relational Software, Inc.), que implemento SQL antes que el propio IBM. El instituto de estandarizacion ANSI publico el primer estandar oficial en 1986, conocido como SQL-86, seguido al ano siguiente por ISO. SQL-89 fue una revision menor, pero SQL-92 marco un hito: definio tres niveles de conformidad (Entry, Intermediate, Full) y estandarizo caracteristicas como JOINs, funciones de agregacion y manejo de cursores. SQL:1999 incorporo tipos definidos por el usuario, triggers, recursividad y el lenguaje procedural SQL/PSM. SQL:2003 introdujo las funciones de ventana, las columnas generadas y el soporte para XML. SQL:2006 se centro en XML, SQL:2008 en los claros dejados por versiones anteriores (TRUNCATE, INSTEAD OF triggers), y SQL:2011 trajo el manejo de datos temporales (_period tables_). SQL:2016 incorporo soporte nativo para JSON, row pattern matching y funciones polimorficas. La revision mas reciente, SQL:2023, anadio soporte para datos grafo (_property graphs_), SQL/JSON mejorado y nuevas funciones de agregacion y ordenacion (_GREATEST_, _LEAST_, _ANY_VALUE_).

```
1970:  SEQUEL (IBM System R)
1979:  Oracle lanza el primer SGBD SQL comercial
1986:  ANSI SQL-86 (primer estandar)
1987:  ISO SQL-87
1989:  SQL-89 (revision menor)
1992:  SQL-92 (Entry, Intermediate, Full)
1999:  SQL:1999 (tipos definidos, triggers, recursividad)
2003:  SQL:2003 (window functions, XML, columnas generadas)
2006:  SQL:2006 (XML nativo)
2008:  SQL:2008 (TRUNCATE, INSTEAD OF triggers)
2011:  SQL:2011 (datos temporales)
2016:  SQL:2016 (JSON, row pattern matching)
2019:  SQL:2019 (extensiones multidimensionales)
2023:  SQL:2023 (property graphs, SQL/JSON mejorado)
```

PostgreSQL se distingue por ser el SGBD que mas fielmente sigue el estandar SQL. Cada nueva version incorpora caracteristicas definidas en el estandar mucho antes que sus competidores, lo que lo convierte en la referencia para el desarrollo de aplicaciones portables entre distintos motores de bases de datos.

## Subconjuntos del lenguaje SQL

El lenguaje SQL se organiza en cuatro subconjuntos funcionales, cada uno con un proposito especifico dentro del ciclo de vida de los datos.

### DDL (_Data Definition Language_)

Define la estructura de los objetos de la base de datos: tablas, esquemas, indices, vistas, tipos y dominios. Las sentencias fundamentales son `CREATE`, `ALTER`, `DROP` y `TRUNCATE`. El DDL no maneja datos, sino el molde que los contiene, y sus operaciones suelen ser transaccionales en PostgreSQL (pueden deshacerse con `ROLLBACK`).

### DML (_Data Manipulation Language_)

Opera sobre los datos almacenados: `SELECT` para consultar, `INSERT` para anadir, `UPDATE` para modificar y `DELETE` para eliminar. Es el subconjunto mas utilizado en aplicaciones y el que concentra la mayor parte del esfuerzo de optimizacion mediante planes de ejecucion e indices.

### DCL (_Data Control Language_)

Gestiona los permisos de acceso a los objetos mediante `GRANT` y `REVOKE`. Controla quien puede leer, escribir o ejecutar operaciones sobre cada elemento de la base de datos.

### TCL (_Transaction Control Language_)

Administra las transacciones: `BEGIN` inicia una, `COMMIT` la confirma, `ROLLBACK` la deshace y `SAVEPOINT` permite establecer puntos intermedios de restauracion.

## Naturaleza declarativa

SQL es un lenguaje declarativo: el programador especifica _que_ resultado desea obtener, no _como_ obtenerlo. El motor de la base de datos es el responsable de determinar el plan de ejecucion optimo mediante su optimizador de consultas, que evalua estadisticas, indices disponibles y costos relativos para elegir la estrategia mas eficiente.

```sql
SELECT e.nombre, d.nombre_departamento
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id
WHERE e.salario > 50000;
```

En este ejemplo no se indica si debe usarse un `Nested Loop`, un `Hash Join` o un `Merge Join`, ni si conviene recorrer un indice o hacer un barrido secuencial. El optimizador decide por si mismo, y el programador solo expresa la relacion logica entre los datos.

## Convenciones lexicas

SQL no distingue entre mayusculas y minusculas en sus palabras reservadas (`SELECT`, `select` y `Select` son equivalentes), aunque por convencion se escriben en mayusculas para mejorar la legibilidad. Los identificadores (nombres de tablas, columnas, esquemas) se convierten a minusculas a menos que se delimiten con comillas dobles, en cuyo caso el motor los trata de forma sensible a mayusculas (_case-sensitive_). Cada sentencia se cierra con punto y coma (`;`), aunque psql permite omitirlo en sentencias simples.

```sql
SELECT * FROM empleados;  -- correcto
select * from Empleados;  -- equivalente (se busca "empleados" en minusculas)
SELECT * FROM "Empleados";  -- distinto: busca exactamente "Empleados" con mayuscula
```

Los comentarios de una linea se introducen con `--` y los multilinea con `/* ... */`. PostgreSQL extiende ademas el estandar con el operador de concatenacion `||`, el casting con `::` y el operador de asignacion `:=` en PL/pgSQL.

