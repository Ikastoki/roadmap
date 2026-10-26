# El sistema de archivos (Filesystem)

Un sistema de archivos (*filesystem*) es la estructura lógica y los mecanismos que organizan, almacenan, recuperan y gestionan los archivos y directorios en un medio de almacenamiento. Sin un sistema de archivos, los datos en un disco serían una masa indiferenciada de bytes sin estructura ni acceso ordenado. El sistema de archivos proporciona la capa de abstracción que permite a los usuarios y programas trabajar con archivos de forma intuitiva, sin conocer los detalles físicos de cómo se almacenan los datos en el medio.

El diseño de un sistema de archivos es una de las tareas más complejas de la ingeniería de software, ya que debe equilibrar múltiples objetivos a menudo conflictivos: rendimiento, fiabilidad, seguridad, escalabilidad y compatibilidad. Las decisiones de diseño en este nivel afectan directamente al rendimiento del sistema, a la integridad de los datos y a la experiencia del usuario.

## Propósito y funciones

Las funciones principales de un sistema de archivos son:

**Organización:** estructurar los datos en archivos y directorios jerárquicos que faciliten su localización y gestión. El sistema de archivos define cómo se nombran los archivos, cómo se agrupan en directorios y cómo se establecen las relaciones entre ellos.

**Almacenamiento:** asignar espacio en el medio de almacenamiento para los datos de cada archivo. El sistema de archivos decide dónde se escriben los bytes físicamente, gestionando los bloques libres y ocupados del disco.

**Recuperación:** proporcionar mecanismos para localizar y leer los datos de un archivo cuando un programa o usuario lo solicita. El sistema de archivos traduce la ruta lógica de un archivo en las direcciones físicas del medio de almacenamiento.

**Integridad:** proteger los datos contra corrupción mediante mecanismos como el journaling, las copias de seguridad y la verificación de checksums. Un buen sistema de archivos puede recuperarse de fallos de energía o errores de hardware sin pérdida de datos.

**Seguridad:** controlar el acceso a los archivos y directorios mediante permisos, cifrado y mecanismos de autenticación. El sistema de archivos garantiza que solo los usuarios autorizados puedan acceder a los datos.

## Mapeo de información lógica a bloques físicos

El sistema de archivos divide el medio de almacenamiento en bloques de tamaño fijo (típicamente 4 KB) y gestiona la asignación de bloques a archivos. Cuando un archivo se escribe, el sistema de archivos selecciona bloques libres en el disco y almacena los datos del archivo en esos bloques, registrando en los metadatos la correspondencia entre las posiciones lógicas del archivo y las posiciones físicas de los bloques.

Existen varias estrategias de asignación de bloques:

**Asignación contigua:** cada archivo se almacena en una secuencia continua de bloques. Ofrece el mejor rendimiento de lectura secuencial pero dificultad para encontrar espacio libre cuando el disco está parcialmente lleno.

**Asignación enlazada:** cada bloque del archivo contiene un puntero al siguiente bloque, formando una lista enlazada. Permite utilizar bloques dispersos por el disco pero degrada el rendimiento de lectura aleatoria.

**Asignación indexada:** un bloque de índice almacena las direcciones de todos los bloques del archivo. Permite acceso aleatorio eficiente pero requiere espacio adicional para los índices. Los inodos de Unix utilizan esta estrategia.

> La elección de la estrategia de asignación de bloques afecta significativamente al rendimiento. Los SSD, al no tener partes mecánicas, no sufren la penalización de accesos aleatorios que afecta a los HDD, lo que permite estrategias de asignación más flexibles sin degradación significativa del rendimiento.

