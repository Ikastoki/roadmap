# SGBD y PostgreSQL

## Que es un SGBD

Un Sistema de Gestion de Bases de Datos (SGBD, o DBMS por sus siglas en ingles) es una capa de software que actua como intermediario entre los datos fisicamente almacenados y los usuarios o aplicaciones que necesitan acceder a ellos. No se limita a leer y escribir archivos: el SGBD es el responsable de implementar todas las funciones que distinguen a una base de datos de un simple almacenamiento de archivos.

Las funciones principales de un SGBD abarcan varias capas de responsabilidad:

**Gestion del almacenamiento:** organiza los datos en estructuras de disco eficientes (paginas, bloques, segmentos), gestiona la asignacion de espacio y optimiza el acceso fisico mediante indices, mapas de bits y otras estructuras de datos.

**Control de concurrencia:** permite que multiples usuarios y aplicaciones accedan a los datos simultaneamente sin interferencias, implementando mecanismos de bloqueo, versionado (MVCC) y transacciones con niveles de aislamiento configurables.

**Gestor de transacciones:** garantiza las propiedades ACID en cada operacion, manteniendo un registro de escritura anticipada (WAL) que permite deshacer transacciones abortadas y recuperar el sistema ante fallos.

**Lenguaje de consultas:** proporciona un lenguaje declarativo (tipicamente SQL) para definir, consultar y manipular los datos. El SGBD incluye un optimizador de consultas que transforma las declaraciones SQL en planes de ejecucion eficientes, eligiendo entre distintos algoritmos de join, estrategias de indexacion y metodos de acceso.

**Seguridad y control de acceso:** gestiona la autenticacion de usuarios, la autorizacion de permisos a nivel de base de datos, esquema, tabla, columna e incluso fila (Row-Level Security), y el cifrado de las comunicaciones y los datos en reposo.

**Integridad:** impone restricciones declarativas (claves primarias, foraneas, unicas, check constraints) para garantizar que los datos cumplen las reglas del negocio, independientemente de como se realicen las modificaciones.

**Copias de seguridad y recuperacion:** proporciona herramientas para realizar copias de seguridad (logicasy fisicas) y restaurar la base de datos a un punto anterior, incluyendo la recuperacion point-in-time mediante el archivo de WAL.

## Comparativa de SGBD

El panorama de los SGBD relacionales esta dominado por cinco sistemas que, aunque comparten el modelo relacional y SQL, difieren significativamente en licencia, arquitectura, rendimiento y ecosistema.

### PostgreSQL

- **Licencia:** Open source (PostgreSQL License, similar a MIT/BSD)
- **Plataformas:** Linux, macOS, Windows, BSD
- **Cumplimiento SQL:** SQL:2023, el mas completo del open source
- **Concurrencia:** MVCC (Multi-Version Concurrency Control) con 4 niveles de aislamiento
- **Replicacion:** Streaming síncrona y asincrona, logical replication, cascading
- **Extensiones:** Sistema de extensiones nativo (CREATE EXTENSION), con mas de 300 extensiones disponibles en el repositorio oficial (Pgxn)
- **Casos de uso:** Aplicaciones transaccionales, geospatial (PostGIS), analitica, JSON/NoSQL

### MySQL

- **Licencia:** Open source (GPLv2) con edicion comercial de Oracle
- **Plataformas:** Linux, macOS, Windows
- **Cumplimiento SQL:** Parcial, carece de algunas caracteristicas del estandar (CTE recursivas, window functions hasta version 8.0)
- **Concurrencia:** MVCC con InnoDB, pero MyISAM (tablas planas) aun presente en usos legacy
- **Replicacion:** Binlog-based replication, Group Replication
- **Extensiones:** Limitadas en comparacion con PostgreSQL, sin un sistema de extensiones estandarizado
- **Casos de uso:** Web (LAMP/LEMP stack), WordPress, aplicaciones de lectura intensiva

### Oracle Database

- **Licencia:** Propietaria (comercial, muy costosa)
- **Plataformas:** Linux, Windows, Solaris, AIX
- **Cumplimiento SQL:** SQL:2023, con numerosas extensiones propietarias (PL/SQL)
- **Concurrencia:** MVCC con undo segments, niveles de aislamiento configurables
- **Replicacion:** Oracle Data Guard, GoldenGate, Active Data Guard
- **Extensiones:** RAC (Real Application Clusters), Partitioning, Advanced Security, todas de pago
- **Casos de uso:** Grandes corporaciones, sistemas financieros, ERP, entornos donde el costo no es el factor limitante

### Microsoft SQL Server

- **Licencia:** Propietaria (comercial, con edicion gratuita limitada: SQL Server Express)
- **Plataformas:** Windows (con soporte limitado para Linux desde 2017)
- **Cumplimiento SQL:** SQL:2023, con extensiones propietarias (T-SQL)
- **Concurrencia:** MVCC con version store en tempdb, niveles de aislamiento snapshots
- **Replicacion:** Always On Availability Groups, Replication Services, Log Shipping
- **Extensiones:** SQL Server Integration Services (SSIS), Reporting Services (SSRS), Analysis Services (SSAS)
- **Casos de uso:** Empresas del ecosistema Microsoft, aplicaciones .NET, BI corporativo

### SQLite

- **Licencia:** Open source (dominio publico)
- **Plataformas:** Prácticamente todas (embebido en el proceso, sin servidor)
- **Cumplimiento SQL:** Subconjunto del estandar SQL (sin procedimientos almacenados, sin alter table complejo)
- **Concurrencia:** Bloqueo a nivel de archivo (soporta lecturas concurrentes, escritura exclusiva)
- **Replicacion:** No nativa
- **Extensiones:** Carga dinamica de extensiones, pero sin ecosistema amplio
- **Casos de uso:** Aplicaciones moviles, embebidas, prototipado rapido, testing, dispositivos IoT

## Por que PostgreSQL

PostgreSQL no es simplemente otro SGBD relacional open source; es el resultado de mas de tres decadas de desarrollo academico y comunitario que lo han convertido en el motor de bases de datos relacional mas avanzado del codigo abierto. Varias razones explican por que es la opcion preferida para nuevos proyectos y migraciones.

**Open source sin restricciones:** a diferencia de MySQL, cuyo nucleo es propiedad de Oracle y cuya licencia GPL puede ser conflictiva en entornos corporativos, PostgreSQL se distribuye bajo una licencia tipo MIT/BSD que permite su uso, modificacion y redistribucion sin restricciones, incluso en productos comerciales cerrados.

**Cumplimiento del estandar SQL:** PostgreSQL es el SGBD open source que mas fielmente sigue el estandar SQL publicado por ISO/IEC. Soporta caracteristicas avanzadas que otros sistemas open source tardan anos en implementar: CTE recursivas, window functions, funciones de ventana, tipos de rango, tablas heredadas, particionamiento nativo y exclusion constraints.

**Extensibilidad:** el diseno de PostgreSQL permite que los desarrolladores agreguen nuevos tipos de datos, operadores, funciones de agregacion, metodos de indexacion y lenguajes de procedimientos almacenados sin modificar el nucleo. Extensiones como PostGIS (sistema de informacion geografica), pg_trgm (busqueda por similitud de texto), pgcrypto (cifrado) y pg_partman (particionamiento automatico) demuestran la potencia de este enfoque. Ningun otro SGBD open source ofrece una capacidad de extension comparable sin recurrir a forks del codigo fuente.

**MVCC y concurrencia:** el sistema de control de concurrencia multiversion de PostgreSQL permite que las lecturas nunca bloqueen a las escrituras ni viceversa. Cada transaccion ve una snapshot coherente de los datos al momento de iniciar su consulta, lo que elimina virtualmente los problemas de contención en cargas de trabajo mixtas de lectura y escritura.

**Ecosistema y comunidad:** PostgreSQL cuenta con una comunidad de desarrollo activa (PostgreSQL Global Development Group) que publica una version mayor anual, documentacion exhaustiva, listas de correo tecnicas y conferencias internacionales (PgConf). El ecosistema incluye herramientas de alta calidad como pgAdmin, DBeaver, PgBouncer (pooling de conexiones), Patroni (alta disponibilidad) y pgBadger (analisis de logs).

**Coste total de propiedad:** al ser gratuito y ejecutarse sobre hardware commodity o cloud, PostgreSQL elimina los costes de licencias que pueden convertir a Oracle o SQL Server en opciones prohibitivas. Su eficiencia en el uso de recursos permite obtener rendimiento competitivo con configuraciones hardware modestas, y su portabilidad evita la dependencia de un unico proveedor de cloud.

