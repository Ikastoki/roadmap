# Arquitectura de PostgreSQL

PostgreSQL implementa una arquitectura de procesos cooperantes donde el servidor se compone de multiples procesos que comparten memoria y se coordinan para gestionar conexiones, realizar operaciones de E/S y mantener la integridad de los datos. A diferencia de los modelos basados exclusivamente en hilos (como MySQL con NDB o SQL Server), PostgreSQL utiliza un proceso por conexion: cada cliente que se conecta provoca que el proceso postmaster cree un nuevo proceso hijo independiente. Este diseno sacrifica cierta eficiencia en memoria respecto a un modelo multi-hilo, pero gana en aislamiento y robustez: un fallo en un proceso backend nunca afecta a las demas conexiones.

La arquitectura de PostgreSQL se divide en tres grandes bloques: el sistema de procesos (postmaster, backend, procesos auxiliares), la memoria compartida (shared buffers, WAL buffer, plan cache) y el sistema de E/S (WAL, tablas, indices, vacuum). Comprender como interactuan estos componentes es esencial para diagnosticar problemas de rendimiento y configurar el servidor de forma optima.

## Postmaster

El proceso postmaster (o postgres, que es el nombre del binario) es el primer proceso que se inicia al arrancar el servidor. Es el padre de todos los demas procesos y su funcion principal es escuchar en el puerto configurado (por defecto 5432) las conexiones entrantes de los clientes. Cuando un cliente solicita una conexion, el postmaster realiza la autenticacion y, si es exitosa, hace un fork de si mismo para crear un proceso backend dedicado que gestionara esa sesion a partir de ese momento.

Ademas de la gestion de conexiones, el postmaster supervisa el estado de los procesos hijos: si un backend termina inesperadamente, el postmaster detecta la senyal SIGCHLD y, en funcion del tipo de fallo, puede reiniciar el sistema completo o simplemente registrar el incidente. Tambien es responsable de iniciar y detener los procesos auxiliares durante el arranque y apagado del servidor.

El postmaster no ejecuta consultas SQL ni accede a los datos directamente. Su existencia esta ligada al ciclo de vida del servidor: cuando termina, todos los procesos hijos terminan con el.

## Procesos backend

Cada conexion de cliente es atendida por un proceso backend independiente. Este proceso es un fork del postmaster que hereda su espacio de memoria, pero que a partir de ese momento opera con su propia pila de llamadas y su propia asignacion de memoria privada. El backend recibe las declaraciones SQL del cliente, las analiza sintactica y semanticamente, genera un plan de ejecucion optimizado, lo ejecuta y devuelve los resultados.

Los procesos backend se comunican con el cliente a traves de un socket TCP o un socket Unix, utilizando un protocolo de mensajes de capa 7 disenado especificamente para PostgreSQL. Cada backend mantiene su propio estado de transaccion, su cache de planes (plan cache) y su memoria local para ordenacion y agregaciones (work_mem).

> El modelo un-proceso-por-conexion implica que el limite maximo de conexiones (max_connections) debe configurarse teniendo en cuenta la memoria disponible. Cada backend consume memoria aunque este inactivo, y si el numero de conexiones es muy alto, la sobrecarga puede degradar el rendimiento. En escenarios con miles de conexiones simultaneas, se utiliza un connection pooler como PgBouncer para multiplexar las conexiones reales.

## Shared buffers

La memoria compartida es la region de memoria accesible por todos los procesos de PostgreSQL. Su componente mas importante es el pool de shared buffers, que actua como cache de paginas de datos. Cuando un proceso backend necesita leer una pagina (tipicamente 8 KB), primero la busca en los shared buffers. Si no esta presente, la pagina se lee desde disco y se almacena en la cache para futuros accesos.

El tamano de shared_buffers es el parametro de configuracion mas influyente en el rendimiento de PostgreSQL. Un valor tipico es el 25 % de la RAM total del servidor, aunque valores superiores pueden ser beneficiosos en sistemas con mucha memoria y cargas de trabajo con lecturas repetitivas. Si shared_buffers es demasiado pequeno, las paginas se descartan de la cache con frecuencia y el servidor pasa mas tiempo leyendo de disco (cache miss). Si es demasiado grande, el tiempo de recuperacion ante un fallo aumenta y la memoria podria competir con el cache del sistema operativo.

PostgreSQL delega parte de la gestion de cache al sistema operativo: es habitual que el sistema operativo mantenga en su cache de pagina muchas de las paginas que PostgreSQL acaba de leer, por lo que el tamano optimo de shared_buffers suele ser menor del esperado. La configuracion efectiva depende del patron de acceso de la aplicacion y del perfil de E/S del hardware.

## Write-Ahead Logging (WAL)

El WAL (Write-Ahead Log, registro de escritura anticipada) es el mecanismo que garantiza la durabilidad de las transacciones. El principio es simple: antes de aplicar cualquier cambio a las paginas de datos en disco, PostgreSQL registra el cambio en el WAL. Si el sistema se cae inesperadamente, en el reinicio se reproduce el WAL desde el ultimo punto de control (checkpoint) para restaurar el estado consistente de los datos.

Cada transaccion que realiza cambios genera registros WAL que se escriben secuencialmente en archivos ubicados en el directorio pg_wal (antes pg_xlog). La escritura es secuencial, lo que permite aprovechar el rendimiento de los discos en operaciones de escritura secuencial, mucho mas rapidas que las escrituras aleatorias. Cuando una transaccion se confirma (COMMIT), el servidor espera a que todos los registros WAL de esa transaccion se hayan escrito en disco antes de notificar al cliente el exito de la operacion.

El WAL no solo sirve para la recuperacion ante fallos: tambien es la base de la replicacion por streaming. Un servidor standby conecta al primario y recibe continuamente los registros WAL generados, reproduciendolos en su propia instancia para mantener una copia actualizada de los datos. Esta funcionalidad permite implementar alta disponibilidad con conmutacion por error en segundos.

> El parametro wal_level determina cuanta informacion se registra en el WAL. El nivel replica (por defecto) incluye la informacion necesaria para replicacion logica y fisica. El nivel logical anade datos adicionales para la replicacion logica (decodificacion de cambios a nivel de fila). El nivel minimal solo registra lo necesario para recuperacion ante fallos, pero impide el uso de replicacion.

## Autovacuum

El autovacuum es el proceso de mantenimiento que limpia los dead tuples generados por las operaciones de actualizacion y eliminacion. En PostgreSQL, cuando se actualiza una fila, el motor no modifica la fila existente: marca la version anterior como muerta (dead tuple) y crea una nueva version. Con el tiempo, estas filas muertas se acumulan y consumen espacio en disco, ademas de degradar el rendimiento de las lecturas porque el motor tiene que recorrerlas para determinar su visibilidad.

El proceso autovacuum se ejecuta en segundo plano como un proceso hijo del postmaster. Despierta periodicamente (configurable mediante autovacuum_naptime, por defecto 1 minuto) y consulta las estadisticas del sistema para identificar las tablas que necesitan vacuum. Cuando el numero de dead tuples supera un umbral (autovacuum_vacuum_threshold mas un porcentaje de filas vivas), lanza un worker de autovacuum dedicado a esa tabla.

Ademas de limpiar dead tuples, el autovacuum actualiza las estadisticas del planificador (equivalente a ejecutar ANALYZE) cuando detecta que el porcentaje de filas modificadas supera el umbral de autovacuum_analyze_threshold. Esto asegura que el planificador de consultas disponga de estadisticas actualizadas sin intervencion manual.

El autovacuum tambien previene el problema de wraparound de transaction IDs (XID). PostgreSQL utiliza identificadores de transaccion de 32 bits, lo que significa que el espacio de XID se agota tras aproximadamente 4000 millones de transacciones. Para evitar que esto ocurra, el autovacuum ejecuta vacuum en modo agresivo cuando se acerca el limite, incluso si eso significa consumir recursos significativos del sistema.

## MVCC (Multi-Version Concurrency Control)

MVCC es el pilar sobre el que se asienta la gestion de concurrencia en PostgreSQL. A diferencia de los sistemas que utilizan bloqueos de lectura para evitar conflictos, MVCC permite que multiples transacciones accedan a los mismos datos simultaneamente sin bloquearse entre si, manteniendo cada transaccion una vista consistente de los datos segun su nivel de aislamiento.

El mecanismo funciona asignando un XID (Transaction ID) unico a cada transaccion que realiza cambios. Cuando una fila se inserta, almacena el XID de la transaccion que la creo (xmin). Cuando una fila se elimina logicamente, almacena el XID de la transaccion que la elimino (xmax). Cada transaccion, al leer una fila, compara estos XIDs con su propio contexto para determinar si la fila es visible para ella.

Las reglas de visibilidad son las siguientes:

1. Una fila es visible si su xmin fue cometido antes de que comenzara la transaccion actual y su xmax es 0 (no ha sido eliminada) o el xmax fue cometido despues de que comenzara la transaccion actual.
2. Una fila no es visible si su xmin fue cometido despues de que comenzara la transaccion actual (aun no existia cuando la transaccion empezo) o si su xmax fue cometido antes de que comenzara la transaccion actual (ya habia sido eliminada).

Esta logica implica que los writers nunca bloquean a los readers: una transaccion que escribe puede estar modificando filas mientras otra transaccion las lee, porque cada una ve la version de los datos que le corresponde segun su XID de visibilidad. Los writers pueden bloquearse entre si si intentan modificar la misma fila simultaneamente, pero este bloqueo es de corta duracion y se resuelve con el primer COMMIT o ROLLBACK.

> El tamaño maximo de XID es de 2^32, aproximadamente 4000 millones de identificadores. Cuando un XID se agota y da la vuelta (wraparound), las reglas de visibilidad se romperian porque un XID futuro pareceria pasado. Para evitarlo, PostgreSQL fuerza un vacuum agresivo (autovacuum o vacuum manual con FREEZE) que marca las filas como frozen, asignandoles un XID especial (FrozenXID, valor 2) que es visible para todas las transacciones.

