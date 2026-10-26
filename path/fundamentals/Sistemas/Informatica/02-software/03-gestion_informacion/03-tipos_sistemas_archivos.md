# Tipos de sistemas de archivos

Los sistemas de archivos varían enormemente según el sistema operativo, el medio de almacenamiento y los requisitos de la aplicación. Cada sistema de archivos fue diseñado para resolver problemas específicos: NTFS prioriza la compatibilidad con Windows y las necesidades empresariales, ext4 optimiza para Linux con fiabilidad y rendimiento, APFS está diseñado para los dispositivos Apple con almacenamiento Flash, y FAT32 prioriza la compatibilidad universal a costa de funcionalidades avanzadas.

Comprender las características, fortalezas y limitaciones de cada sistema de archivos es esencial para tomar decisiones informadas sobre cómo formatear y gestionar los medios de almacenamiento.

## NTFS

NTFS (*New Technology File System*) es el sistema de archivos estándar de Windows desde Windows NT (1993). Ofrece una combinación de funcionalidades avanzadas que lo hacen adecuado tanto para uso doméstico como empresarial:

**Journaling:** NTFS mantiene un registro de transacciones pendientes que permite recuperar archivos y directorios en caso de fallo del sistema, reduciendo significativamente el riesgo de corrupción de datos.

**Permisos y seguridad:** NTFS soporta listas de control de acceso (ACL) que permiten definir permisos granulares para usuarios y grupos sobre archivos y directorios individuales.

**Compresión y cifrado:** NTFS permite comprimir archivos individualmente para ahorrar espacio, y soporta cifrado mediante EFS (*Encrypting File System*).

**Tamaño máximo de archivo:** 16 TB teóricos (256 TB con formats específicos). Límites prácticos de 4 TB para la mayoría de implementaciones.

**Límites conocidos:** NTFS no es nativamente compatible con macOS (solo lectura) y tiene soporte limitado en Linux (escritura experimental). Su sobrecarga de metadatos lo hace menos eficiente para archivos muy pequeños.

## ext4

ext4 (*fourth extended filesystem*) es el sistema de archivos estándar de la mayoría de distribuciones Linux. Es una evolución madura de ext2 y ext3, con mejoras significativas en rendimiento, fiabilidad y capacidad:

**Journaling:** heredado de ext3, ext4 mantiene un diario de transacciones que garantiza la integridad de los datos ante fallos del sistema.

**Allocación diferida (*delayed allocation*):** retrasa la asignación de bloques hasta el momento de la escritura, permitiendo al kernel agrupar mejor los datos y reducir la fragmentación.

**Extentos:** utiliza extentos (rangos continuos de bloques) en lugar de punteros individuales, mejorando significativamente el rendimiento de archivos grandes y reduciendo el espacio consumido por metadatos.

**Tamaño máximo de archivo:** 16 TB. Tamaño máximo del sistema de archivos: 1 EB (exabyte).

**Snapshots:** ext4 soporta copias instantáneas del sistema de archivos mediante herramientas como `fstransform` o LVM.

## APFS

APFS (*Apple File System*) es el sistema de archivos diseñado por Apple para macOS, iOS, iPadOS, watchOS y tvOS. Fue lanzado en 2017 como sucesor de HFS+ y está optimizado para almacenamiento Flash y SSD:

**Optimización para Flash:** APFS está diseñado específicamente para medios de estado sólido, minimizando las escrituras innecesarias y optimizando la distribución de datos para maximizar la vida útil del SSD.

**Snapshots instantáneos:** APFS permite crear copias casi instantáneas del sistema de archivos sin duplicar los datos, utilizando técnica *copy-on-write*.

**Cifrado nativo:** APFS soporta cifrado de un solo extremo o con clave compartida, integrado de forma transparente en el sistema.

**Clonación de archivos:** APFS puede crear copias de archivos casi instantáneas que comparten los mismos bloques de datos hasta que uno de ellos se modifica.

**Límites conocidos:** APFS no es compatible con Windows ni Linux de forma nativa, lo que limita su uso a ecosistemas Apple.

## FAT32

FAT32 (*File Allocation Table 32*) es uno de los sistemas de archivos más antiguos y compatibles, introducido en 1996. Su diseño simple y amplia compatibilidad lo hacen indispensable para intercambio de datos entre diferentes sistemas operativos:

**Compatibilidad universal:** FAT32 se puede leer y escribir en Windows, macOS, Linux, consolas de juegos, cámaras, reproductores multimedia y prácticamente cualquier dispositivo con puerto USB.

**Simplicidad:** su estructura es simple y ligera, lo que lo hace adecuado para dispositivos con recursos limitados.

**Límites significativos:**
- Tamaño máximo de archivo: 4 GB (un solo archivo no puede superar esta cifra).
- Tamaño máximo del sistema de archivos: 2 TB (Windows limita la creación de volúmenes FAT32 a 32 GB).
- Sin journaling: no tiene protección contra corrupción por fallos del sistema.
- Sin permisos: no soporta ACL ni permisos por usuario.

> FAT32 sigue siendo el sistema de archivos recomendado para dispositivos USB y tarjetas de memoria que deben funcionar en múltiples plataformas. Para discos internos donde la compatibilidad entre sistemas no es necesaria, NTFS (Windows), ext4 (Linux) o APFS (macOS) son opciones superiores.

