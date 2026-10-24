# Modelos de datos

## Modelo relacional

El modelo relacional, propuesto por Edgar F. Codd en 1970 mientras trabajaba en IBM, se fundamenta en la teoria de conjuntos y la logica de predicados. Su proposicion era radical: los datos debian organizarse en relaciones (tablas) independientes de la implementacion fisica, de modo que los usuarios interactuaran con ellos mediante un lenguaje de alto nivel sin preocuparse por la estructura de almacenamiento subyacente.

Una base de datos relacional se compone de tablas, tambien denominadas relaciones, que siguen una estructura rigida de filas y columnas.

**Tablas (relaciones):** representan entidades del mundo real (usuarios, pedidos, productos). Cada tabla tiene un conjunto fijo de columnas definidas en el momento de su creacion, lo que constituye el esquema de la tabla.

**Filas (tuplas):** cada fila es una ocurrencia concreta de la entidad. Una fila contiene un valor para cada columna definida en el esquema, aunque algunos valores pueden ser nulos si la columna lo permite.

**Columnas (atributos):** definen las propiedades de la entidad. Cada columna tiene un nombre unico dentro de la tabla y un tipo de dato que restringe los valores que puede albergar (entero, texto, fecha, etc.).

El poder del modelo relacional no reside en la organizacion individual de las tablas, sino en la capacidad de relacionarlas entre si mediante claves:

**Clave primaria (Primary Key - PK):** columna o combinacion de columnas que identifica de forma unica cada fila de una tabla. Garantiza que no existen dos filas con el mismo valor de clave primaria, lo que constituye una restriccion de unicidad e integridad de entidad. Por convencion, suele anadirse una columna `id` de tipo entero autoincremental, aunque puede ser cualquier conjunto de columnas que cumpla la unicidad.

**Clave foranea (Foreign Key - FK):** columna o combinacion de columnas cuyos valores referencian la clave primaria de otra tabla. La FK establece una relacion entre dos tablas y garantiza la integridad referencial: no puede existir un valor de FK que no tenga correspondencia en la tabla referenciada (a menos que se permita NULL). Las acciones referenciales (`ON DELETE CASCADE`, `ON UPDATE SET NULL`, etc.) determinan que ocurre con las filas hijas cuando se modifica o elimina la fila padre.

**Tipos de relaciones entre entidades:**
- **Uno a muchos (1:N):** la relacion mas frecuente. Una fila de la tabla A se asocia con varias filas de la tabla B. Ejemplo: un cliente puede tener muchos pedidos. Se implementa incluyendo la PK de la tabla del lado "uno" como FK en la tabla del lado "muchos".
- **Muchos a muchos (N:M):** dos entidades donde cada fila de A puede relacionarse con varias de B y viceversa. Requiere una tabla intermedia que contenga las FK de ambas tablas. Ejemplo: estudiantes y cursos, donde la tabla `matriculas` contiene `estudiante_id` y `curso_id`.
- **Uno a uno (1:1):** cada fila de A se asocia con exactamente una fila de B. Se implementa incluyendo la FK en la tabla secundaria con una restriccion UNIQUE. Ejemplo: usuario y datos de facturacion.

## Modelo NoSQL

El termino NoSQL (Not Only SQL) agrupa una familia de sistemas de bases de datos que surgieron a mediados de la decada de 2000 para cubrir casos de uso donde el modelo relacional presentaba limitaciones practicas, especialmente en escalabilidad horizontal, manejo de datos semiestructurados y rendimiento en operaciones especificas. No existe un unico modelo NoSQL, sino cuatro grandes categorias con propiedades muy distintas:

**Bases de datos documentales:** almacenan documentos en formatos semiestructurados (JSON, BSON, XML). Cada documento es autocontenido y puede tener una estructura diferente del resto, lo que se conoce como esquemas flexibles o "schemaless". La unidad de datos suele albergar toda la informacion necesaria para una entidad, evitando joins costosos mediante la desnormalizacion intencionada. MongoDB, CouchDB y Firestore son los representantes mas conocidos. Son especialmente adecuadas para catalogos de productos, sistemas de gestion de contenidos y aplicaciones donde el esquema evoluciona rapidamente.

**Bases de datos clave-valor:** representan el modelo mas simple y rapido. Funcionan como un diccionario distribuido donde cada clave unica se asocia a un valor que puede ser cualquier secuencia de bytes. La unica operacion eficiente es la recuperacion por clave exacta; las consultas por valor requieren escaneos completos. Redis (en memoria), DynamoDB y Riak destacan en esta categoria. Su caso de uso natural son las sesiones de usuario, cachas, colas de mensajes y contadores.

**Bases de datos orientadas a columnas:** almacenan los datos agrupados por columna y no por fila, lo que permite leer solo los atributos necesarios sin cargar filas completas. Esto las hace extremadamente eficientes para consultas analiticas y agregaciones sobre grandes volumenes de datos. Cassandra, HBase y ScyllaDB son ejemplos destacados. Se utilizan en aplicaciones de IoT, series temporales y analisis de eventos.

**Bases de datos de grafos:** modelan los datos como nodos (entidades) conectados por aristas (relaciones), donde ambas pueden tener propiedades. La navegacion por las relaciones es la operacion fundamental, y su eficiencia no se degrada con la profundidad de la consulta como ocurriria con multiples joins en SQL. Neo4j, Amazon Neptune y ArangoDB son los mas populares. Son ideales para motores de recomendacion, deteccion de fraudes, redes sociales y analisis de dependencias.

## Cuando usar cada modelo

No existe un modelo universalmente superior; la eleccion depende de la naturaleza de los datos, los patrones de acceso y los requisitos operativos.

**Elegir modelo relacional cuando:**
- Los datos tienen una estructura bien definida y estable.
- Se requieren garantias ACID estrictas (transacciones bancarias, ERP, sistemas de facturacion).
- Las consultas implican relaciones complejas entre multiples entidades.
- La integridad referencial debe estar garantizada a nivel de base de datos.
- El equipo tiene experiencia en SQL y modelado relacional.
- Se necesita compatibilidad con herramientas de BI, reporting y ORMs estandar.

**Elegir modelo documental cuando:**
- Los datos son semiestructurados o su esquema evoluciona con frecuencia.
- Las entidades son autocontenidas y las relaciones entre ellas son secundarias.
- La aplicacion requiere escalabilidad horizontal desde el diseno.
- Se prefiere desnormalizar para evitar joins y optimizar lecturas.
- El equipo trabaja con lenguajes orientados a objetos y JSON es el formato nativo de intercambio.

**Elegir modelo clave-valor cuando:**
- La aplicacion necesita latencias de microsegundos para accesos por clave.
- Los datos son simples o se almacenan como objetos opacos (sesiones, cachas).
- La escalabilidad horizontal es critica y la consistencia eventual es aceptable.
- No se requieren consultas complejas ni joins.

**Elegir modelo de columnas cuando:**
- Las consultas son predominantemente analiticas y leen pocas columnas de muchas filas.
- Los datos tienen naturaleza temporal o de series temporales.
- Se manejan petabytes de datos y se necesita compresion agresiva.
- La carga de trabajo es intensiva en escritura con pocas actualizaciones.

**Elegir modelo de grafos cuando:**
- Las relaciones entre entidades son tan importantes como las propias entidades.
- Las consultas recorren multiples niveles de profundidad en las relaciones.
- Se implementan funcionalidades como recomendaciones, deteccion de fraudes o analisis de redes.
- El modelo relacional requeriria una cantidad excesiva de joins con rendimiento impredecible.

En la practica, muchas organizaciones adoptan arquitecturas poliglota de persistencia (polyglot persistence): utilizan el modelo relacional como sistema de registro principal para datos transaccionales y anaden bases de datos NoSQL para cargas de trabajo especificas que se benefician de otro modelo. PostgreSQL, como base de datos relacional, ofrece ademas soporte nativo para JSON, arrays y tipos geometricos, lo que permite cubrir parte del espectro NoSQL sin salir del mismo sistema.

