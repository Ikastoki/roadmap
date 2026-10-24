# Que es NoSQL

El termino NoSQL agrupa una familia de sistemas de gestion de bases de datos que se apartan del modelo relacional clasico (SQL) para abordar problemas que las bases de datos tradicionales resuelven con dificultad: volumenes masivos de datos, esquemas que evolucionan constantemente, cargas de trabajo con alta concurrencia y requisitos de escalamiento horizontal. NoSQL no significa "no SQL" literalmente, sino "not only SQL": reconoce que el modelo relacional sigue siendo optimo para muchos escenarios, pero no para todos.

## Origen y motivacion

A finales de la decada de 2000, empresas como Google, Amazon y Facebook enfrentaron problemas de escalabilidad que las bases de datos relacionales no podian resolver de forma economica. El almacenamiento y procesamiento de petabytes de datos distribuidos en miles de servidores exigia un modelo diferente. De ahi nacen sistemas como BigTable (Google), Dynamo (Amazon) y Cassandra (Facebook), que sacrifican propiedades clasicas de las bases de datos relacionales a cambio de escalabilidad y disponibilidad.

## Limitaciones de SQL ante Big Data

Las bases de datos relacionales fueron disenadas para entornos donde el esquema es conocido de antemano, los datos cabe en un solo servidor y las transacciones requieren garantias estrictas. En los entornos modernos aparecen limitaciones significativas:

- **Escalabilidad vertical**: la unica forma de escalar un RDBMS clasico es mejorar el hardware (mas CPU, mas RAM, discos mas rapidos). Llega un punto donde el coste es prohibitivo o el hardware simplemente no existe.
- **Esquema rigido**: anyadir una columna implica migraciones, downtime y coordinacion entre equipos. En entornos agiles esto se convierte en un cuello de botella.
- **Cuellos de botella en escritura**: las garantias ACID imponen bloqueos y mecanismos de sincronizacion que limitan el rendimiento en escritura bajo alta concurrencia.
- **Dificultad para escalar lecturas**: aunque existan replicas de lectura, la consistencia fuerte obliga a coordinar los nodos, lo que introduce latencia.

> NoSQL no reemplaza a SQL: lo complementa. Muchas arquitecturas modernas usan ambos modelos simultaneamente (bases de datos poliglotas).

## ACID vs BASE

La principal diferencia filosofica entre SQL y NoSQL reside en las propiedades de transaccionalidad que ofrecen.

### ACID (SQL)

- **Atomicity** (Atomicidad): una transaccion se ejecuta completamente o no se ejecuta en absoluto.
- **Consistency** (Consistencia): los datos siempre cumplen las restricciones definidas (claves foraneas, unicidad, tipos).
- **Isolation** (Aislamiento): las transacciones concurrentes no interfieren entre si.
- **Durability** (Durabilidad): una vez commiteada, la transaccion persiste aunque el sistema falle.

ACID garantiza un comportamiento predecible y fiable, pero impone un coste en rendimiento y disponibilidad. Es la opcion correcta cuando la integridad de los datos es critica: sistemas bancarios, reservas, facturacion.

### BASE (NoSQL)

- **Basically Available**: el sistema responde siempre, incluso si parte de los datos no esta disponible o es inconsistente temporalmente.
- **Soft state**: el estado del sistema puede cambiar sin intervencion externa debido a la consistencia eventual.
- **Eventual consistency**: si no hay nuevas escrituras, todos los nodos convergeran al mismo estado con el tiempo.

BASE relaja las garantias de consistencia a cambio de disponibilidad y tolerancia a particiones (teorema CAP). Es la opcion adecuada cuando la disponibilidad y la escalabilidad priman sobre la consistencia inmediata: redes sociales, catalogos de productos, analitica en tiempo real.

> El teorema CAP demuestra que un sistema distribuido solo puede garantizar dos de tres propiedades: consistencia, disponibilidad y tolerancia a particiones. NoSQL suele priorizar disponibilidad y tolerancia a particiones (AP), mientras que SQL prioriza consistencia y disponibilidad (CA).

## Cuando usar SQL vs NoSQL

SQL es la opcion correcta cuando:

- Los datos tienen relaciones complejas y se necesitan joins, claves foraneas y transacciones ACID.
- El esquema es estable y predecible.
- Se requiere consistencia inmediata (lecturas siempre ven la ultima escritura).
- El volumen de datos cabe en un solo servidor o se escala verticalmente.

NoSQL es mas adecuado cuando:

- Los datos tienen un esquema que cambia frecuentemente o es impredecible.
- Se necesita escalar horizontalmente repartiendo datos entre multiples servidores.
- La aplicacion maneja millones de lecturas/escrituras por segundo.
- Los datos son mayoritariamente denormalizados y no requieren joins complejos.
- La disponibilidad es critica y se tolera consistencia eventual.

> En la practica, arquitecturas modernas combinan ambos enfoques: un servicio de catalogo puede usar MongoDB para flexibilidad de esquema, mientras que el modulo de pagos usa PostgreSQL para garantias transaccionales.

