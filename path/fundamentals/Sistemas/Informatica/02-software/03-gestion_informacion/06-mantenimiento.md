# Operadores y mantenimiento

El mantenimiento de los sistemas de archivos es una actividad fundamental para garantizar la fiabilidad, el rendimiento y la integridad de los datos a lo largo del tiempo. Los sistemas de archivos, al igual que cualquier otro componente de un sistema informático, requieren atención periódica para funcionar de forma óptima. La fragmentación, los errores de consistencia y la acumulación de datos obsoletos son problemas que, si no se gestionan, pueden degradar significativamente el rendimiento del sistema y poner en riesgo la integridad de los datos.

## Fragmentación

La fragmentación ocurre cuando los bloques de datos de un archivo se distribuyen por diferentes zonas del medio de almacenamiento en lugar de estar contiguos. En un disco duro mecánico, la fragmentación provoca que los cabezales deban desplazarse físicamente entre diferentes posiciones del platillo para leer un solo archivo, aumentando la latencia de acceso.

La fragmentación se produce naturalmente a medida que el usuario crea, modifica y elimina archivos. Cuando un archivo crece más allá del espacio contiguo disponible, el sistema de archivos asigna bloques adicionales en otra parte del disco, fragmentando el archivo. La eliminación de archivos crea huecos en el disco que el sistema de archivos reutiliza parcialmente, contribuyendo a la fragmentación.

Los SSD no se ven afectados por la fragmentación en términos de rendimiento, ya que su latencia de acceso es prácticamente independiente de la posición física de los datos. Sin embargo, la fragmentación en SSD puede provocar un aumento de las escrituras (ya que el sistema de archivos necesita escribir datos en múltiples ubicaciones) y un mayor consumo de metadatos.

## Desfragmentación

La desfragmentación es el proceso de reorganizar los bloques de datos en el disco para que los archivos estén contiguos, reduciendo los desplazamientos de los cabezales en HDD y mejorando el rendimiento de lectura secuencial.

En Windows, la herramienta de desfragmentación incorporada (*Optimize Drives*) ejecuta automáticamente tareas de desfragmentación según una programación configurada. En Linux, herramientas como `e4defrag` y `fstransform` permiten desfragmentar sistemas de archivos ext4.

> Los SSD no deben desfragmentarse de forma convencional. Las herramientas modernas de mantenimiento de SSD ejecutan *TRIM* en lugar de desfragmentación: el TRIM informa al SSD qué bloques ya no están en uso, permitiendo al controlador del SSD optimizar internamente la distribución de datos y mejorar el rendimiento de escritura.

## El concepto de *Journaling*

El journaling (*registro por diario*) es un mecanismo de integridad de datos que registra las operaciones pendientes antes de aplicarlas al sistema de archivos. Si el sistema falla durante una operación (por ejemplo, por un corte de energía), el journal contiene la información necesaria para completar o revertir la operación de forma consistente.

El proceso de journaling funciona así:

1. **Preparación:** el sistema de archivos escribe en el journal (una área reservada del disco) una descripción de la operación que va a realizar.
2. **Escritura:** el sistema de archivos aplica los cambios al sistema de archivos principal.
3. **Confirmación:** una vez completada la operación, el sistema de archivos marca la entrada del journal como completada.

Si el sistema falla durante el paso 2, al reiniciar el sistema de archivos revisa el journal y detecta la operación incompleta. Puede entonces completar la operación si los datos ya se escribieron, o revertirla si no, garantizando siempre la consistencia del sistema de archivos.

Los sistemas de archivos con journaling (ext3, ext4, NTFS, APFS) son significativamente más resistentes a la corrupción por fallos del sistema que los sistemas sin journaling (FAT32, ext2). El coste del journaling es una pequeña sobrecarga de rendimiento en escritura, ya que cada operación implica dos escrituras: una al journal y una al sistema de archivos principal.

> La estrategia de journaling varía según el sistema de archivos. ext4 utiliza journaling de metadatos por defecto (solo registra cambios en los inodos y estructuras de directorio, no en los datos), lo que ofrece un buen equilibrio entre integridad y rendimiento. Para máxima integridad, se puede configurar journaling de datos completos, a costa de mayor sobrecarga.

## Copias de seguridad y sistemas de redundancia

Las copias de seguridad son la última línea de defensa contra la pérdida de datos. Ningún mecanismo de integridad (journaling, checksums, RAID) sustituye a una copia de seguridad adecuada, ya que ninguno protege contra fallos catastróficos, errores humanos o ataques maliciosos.

**3-2-1:** la regla de oro del backup recomienda mantener 3 copias de los datos, en 2 medios diferentes, con 1 copia fuera del sitio (offsite). Esta estrategia protege contra fallos de hardware, desastres naturales y robo.

**Copias en la nube:** servicios como Backblaze, Google Drive, OneDrive y Dropbox proporcionan almacenamiento offsite automático. La sincronización automática garantiza que los cambios se reflejen en la nube sin intervención manual, pero no sustituye a un backup completo que incluya versiones históricas.

**Instantáneas (*snapshots*):** las instantáneas permiten capturar el estado del sistema de archivos en un momento dado, permitiendo recuperar archivos eliminados o restaurar el sistema a un estado anterior. ZFS, Btrfs y LVM soportan instantáneas nativas que son rápidas y eficientes en espacio.

**Sistemas de redundancia:** RAID (0, 1, 5, 6, 10) proporciona redundancia a nivel de disco, protegiendo contra fallos de un o varios discos. Sin embargo, RAID no sustituye a un backup: no protege contra errores lógicos, ransomware o desastres que afecten al conjunto completo de discos.

