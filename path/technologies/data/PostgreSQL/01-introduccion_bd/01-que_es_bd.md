# Que es una base de datos

Toda aplicacion que persiste informacion mas alla del tiempo de ejecucion de un proceso necesita un mecanismo de almacenamiento estructurado. Un archivo de texto plano puede bastar para unos pocos registros, pero cuando los datos crecen en volumen, complejidad y concurrencia de acceso, la necesidad de un sistema formal de gestion se vuelve ineludible. Una base de datos (BD) es una coleccion organizada de datos diseñada para almacenar, recuperar y manipular informacion de manera eficiente, segura y concurrente. No es simplemente un repositorio pasivo: es un sistema activo que impone reglas de integridad, gestiona el acceso simultaneo de multiples usuarios y garantiza que los datos permanezcan consistentes incluso ante fallos del sistema.

Las aplicaciones reales no ejecutan consultas directamente sobre archivos del sistema operativo porque ello implicaria que cada desarrollador implementase su propio mecanismo de indexacion, sincronizacion y recuperacion ante fallos. La base de datos abstrae toda esa complejidad y la ofrece como un servicio al que las aplicaciones se conectan mediante lenguajes estandarizados, siendo SQL (Structured Query Language) el mas extendido.

## Tipos de bases de datos

Existen varias clasificaciones de los sistemas de bases de datos, pero la mas significativa desde el punto de vista practico atiende al modelo de datos que emplean. Cada modelo resuelve un conjunto diferente de problemas y ninguno es universalmente superior.

**Bases de datos relacionales (RDBMS):** organizan la informacion en tablas compuestas de filas y columnas, donde cada fila representa una entidad y cada columna un atributo. Las relaciones entre tablas se establecen mediante claves primarias y foraneas. Son el modelo dominante desde los anos 80 y constituyen la opcion por defecto para la mayoria de los sistemas transaccionales. Ejemplos: PostgreSQL, MySQL, Oracle, SQL Server.

**Bases de datos NoSQL:** agrupan a todos aquellos sistemas que no siguen el modelo relacional. Nacieron para cubrir escenarios donde la rigidez del esquema relacional, la escalabilidad vertical o el rendimiento en operaciones especificas resultaban limitantes. Se subdividen en cuatro grandes familias:

- **Documentales:** almacenan datos en documentos semiestructurados (JSON, BSON) donde cada documento puede tener una estructura diferente. La unidad de datos es autocontenida y suele serializarse en un unico nodo de almacenamiento, lo que facilita la escalabilidad horizontal. Ejemplos: MongoDB, CouchDB, Firestore.
- **Clave-valor:** el modelo mas simple posible: un diccionario inmenso donde cada clave se asocia a un unico valor, que puede ser un dato plano o un objeto complejo. Ofrecen latencias extremadamente bajas para accesos por clave exacta. Ejemplos: Redis, DynamoDB, Riak.
- **Orientadas a columnas:** almacenan los datos agrupados por columna en lugar de por fila, lo que las hace especialmente eficientes para consultas analiticas que requieren leer unos pocos atributos de millones de registros. Ejemplos: Apache Cassandra, HBase, ClickHouse.
- **Bases de grafos:** representan los datos como nodos (entidades) y aristas (relaciones), donde ambas pueden tener propiedades. Son ideales para modelos donde las relaciones entre entidades son tan importantes como las propias entidades. Ejemplos: Neo4j, ArangoDB, Amazon Neptune.

**Bases de datos hibridas y multimodelo:** muchos sistemas modernos incorporan soporte para mas de un modelo de datos. PostgreSQL, por ejemplo, permite trabajar con documentos JSON, clave-valor mediante extensiones y datos geograficos, aunque su modelo nativo sea relacional.

## Ventajas sobre archivos planos

Almacenar datos en archivos planos (CSV, JSON, XML) es viable en prototipos y analisis exploratorios, pero presenta limitaciones insalvables en entornos de produccion:

| Aspecto | Archivo plano | Base de datos |
|---|---|---|
| Concurrencia | Sin control de acceso simultaneo; dos procesos escribiendo a la vez corrompen el archivo | Gestion de transacciones concurrentes con aislamiento configurable |
| Integridad | Debe implementarse en cada aplicacion cliente | Restricciones declarativas (claves unicas, integridad referencial, check constraints) |
| Recuperacion | Ninguna; si el archivo se corrompe, los datos se pierden | Mecanismos de recuperacion ante fallos (WAL, puntos de control) |
| Indexacion | Solo busqueda secuencial o indexacion manual | Arboles B+ y otras estructuras optimizadas automaticamente |
| Seguridad | Permisos de fichero del sistema operativo | Control granular a nivel de fila, columna y operacion |
| Consultas complejas | Procesamiento manual con scripts ad hoc | Lenguaje declarativo optimizado por el planificador de consultas |
| Escalabilidad | Limitada por un solo disco y procesador | Escalabilidad vertical, horizontal, replicacion y sharding |

El costo de esta potencia es la complejidad operativa: una base de datos es un proceso servidor que consume recursos de CPU, memoria y almacenamiento, y requiere conocimientos especializados para su administracion y ajuste.

## ACID

El acronimo ACID (Atomicity, Consistency, Isolation, Durability) define las propiedades que debe cumplir una transaccion para garantizar que los datos se manejan de forma fiable incluso en presencia de fallos, concurrencia y errores. No todos los sistemas de bases de datos son ACID-compliant, y la decision de implementarlo o no es una de las diferencias fundamentales entre los modelos relacionales y muchos sistemas NoSQL.

**Atomicidad:** una transaccion se ejecuta en su totalidad o no se ejecuta en absoluto. Si falla cualquier operacion dentro de la transaccion, todas las operaciones anteriores deben deshacerse (rollback). Este principio evita que queden datos a medio escribir, como una transferencia bancaria donde se debita de una cuenta pero no se acredita en la otra.

**Consistencia:** una transaccion lleva la base de datos de un estado valido a otro estado valido, respetando todas las reglas de integridad definidas (claves unicas, restricciones de chequeo, integridad referencial). Si una transaccion viola alguna restriccion, es rechazada en su totalidad.

**Aislamiento:** las transacciones concurrentes se ejecutan como si fueran secuenciales, sin interferir entre si. El nivel de aislamiento (read uncommitted, read committed, repeatable read, serializable) determina que fenomenos de concurrencia se permiten: lecturas sucias, lecturas no repetibles y lecturas fantasma. La mayoria de los sistemas relacionales usan read committed por defecto.

**Durabilidad:** una vez que una transaccion se confirma (commit), sus cambios persisten incluso si el sistema se cae inmediatamente despues. La implementacion tipica implica escribir los cambios en un log de transacciones (WAL - Write-Ahead Log) antes de aplicarlos a las estructuras de datos principales.

> El cumplimiento de ACID tiene un costo en rendimiento debido a la sobrecarga de sincronizacion y registro. Los sistemas NoSQL suelen relajar una o mas de estas propiedades (tipicamente aislamiento y consistencia) para ganar en escalabilidad horizontal y latencia. Este compromiso se conoce como teorema CAP (Consistency, Availability, Partition Tolerance).

## Modelo cliente-servidor

Las bases de datos modernas operan bajo una arquitectura cliente-servidor, donde existe un proceso servidor que gestiona los datos y acepta conexiones desde multiples procesos clientes. El servidor es el unico que tiene acceso directo a los archivos de datos, mientras que los clientes se comunican con el exclusivamente a traves de un protocolo de red.

En el caso de PostgreSQL, el servidor se compone de varios procesos cooperantes: el proceso postmaster, que escucha conexiones entrantes y lanza procesos hijos para atender a cada cliente; los procesos de escritura en segundo plano (background writer, WAL writer, autovacuum); y los procesos auxiliares (checkpointer, stats collector). Cada conexion de cliente da lugar a un nuevo proceso hijo (fork) que gestiona esa sesion de forma independiente, lo que proporciona aislamiento entre sesiones.

El protocolo de comunicacion entre cliente y servidor sigue una secuencia establecida:
1. El cliente inicia una conexion TCP hacia el puerto del servidor (por defecto 5432).
2. El servidor autentica al cliente mediante el metodo configurado (password, scram-sha-256, certificados SSL, etc.).
3. Una vez autenticado, el cliente envia declaraciones SQL y recibe resultados.
4. La conexion se mantiene abierta para sucesivas consultas hasta que el cliente la cierra o se agota el timeout.

Este modelo presenta ventajas claras: centraliza la gestion de datos, facilita la seguridad perimetral (solo el puerto de la base de datos necesita exponerse), permite que multiples aplicaciones escritas en distintos lenguajes accedan a los mismos datos y simplifica las copias de seguridad al concentrar todo el almacenamiento en el servidor.

