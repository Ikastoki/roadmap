# Gestión de directorios y metadatos

La gestión de directorios y metadatos es la capa de organización que permite estructurar, localizar y gestionar archivos de forma eficiente. Los directorios proporcionan una jerarquía lógica que agrupa archivos relacionados, mientras que los metadatos contienen toda la información necesaria para gestionar los archivos sin necesidad de leer su contenido. Comprender estos conceptos es fundamental para trabajar eficazmente con sistemas de archivos y para entender cómo los sistemas operativos organizan la información.

## Estructura jerárquica de directorios (árbol)

La estructura de directorios de un sistema de archivos se organiza en forma de árbol, con la raíz (*root*) en la parte superior y directorios y subdirectorios ramificados hacia abajo. Cada directorio puede contener archivos y otros directorios, creando una jerarquía que permite organizar la información en niveles de categorización.

En sistemas Unix/Linux, la raíz se representa con `/` y todo el sistema parte de ella:

```
/
├── home/
│   ├── usuario1/
│   │   ├── documentos/
│   │   ├── descargas/
│   │   └── imagenes/
│   └── usuario2/
├── etc/
├── var/
├── usr/
└── tmp/
```

En Windows, la estructura comienza con letras de unidad (`C:\`, `D:\`), cada una representando una partición o dispositivo de almacenamiento diferente:

```
C:\
├── Usuarios\
│   ├── Usuario1\
│   │   ├── Documentos\
│   │   ├── Descargas\
│   │   └── Imágenes\
│   └── Usuario2\
├── Archivos de programa\
├── Windows\
└── Temp\
```

Los directorios especiales como `.` (directorio actual) y `..` (directorio padre) permiten navegar la jerarquía de forma relativa. La barra `/` en Unix y la barra invertida `\` en Windows son los separadores de directorio.

## El papel de los Inodos

Un inodo (*index node*) es la estructura de datos que almacena los metadatos de un archivo en sistemas de archivos tipo Unix (ext2, ext3, ext4). Cada archivo o directorio tiene un inodo asociado que contiene:

- **Número de inodo:** identificador numérico único del archivo dentro del sistema de archivos.
- **Tipo:** indica si el archivo es un archivo regular, un directorio, un enlace simbólico, un dispositivo de caracteres, etc.
- **Permisos:** los permisos de lectura, escritura y ejecución para propietario, grupo y otros.
- **Propietario y grupo:** el usuario y grupo que poseen el archivo.
- **Tamaño:** el tamaño del archivo en bytes.
- **Marcas de tiempo:** fecha de creación, último acceso, última modificación y último cambio de metadatos.
- **Punteros a bloques:** las direcciones de los bloques en el disco donde se almacenan los datos del archivo.

> Una decisión clave del diseño de ext4 fue aumentar el número de punteros de bloques en el inodo, permitiendo archivos de hasta 16 TB. Los primeros 12 punteros son directos (apuntan directamente a bloques de datos), mientras que los punteros indirectos apuntan a bloques que contienen más punteros, permitiendo archivos enormes a costa de accesos adicionales al disco.

## Vínculos duros y enlaces simbólicos

Los enlaces proporcionan múltiples formas de acceder al mismo archivo:

**Vínculos duros (*hard links*):** un hard link es una entrada adicional en el directorio que apunta al mismo inodo que el archivo original. Técnicamente, el archivo original es simplemente el primer hard link. Los hard links comparten el mismo inodo, por lo que los cambios en un enlace se reflejan inmediatamente en todos los demás, ya que todos acceden a los mismos datos.

Los hard links tienen limitaciones: no pueden cruzar particiones diferentes (ya que los inodos son únicos dentro de una partición) y no pueden enlazar directorios (por razones de integridad de la estructura de árbol).

**Enlaces simbólicos (*symbolic links* o *symlinks*):** un symlink es un archivo especial que contiene la ruta a otro archivo o directorio. A diferencia de los hard links, los symlink pueden apuntar a archivos en diferentes particiones e incluso a rutas inexistentes (*dangling symlink*).

Los symlink son más flexibles pero tienen un overhead adicional: cada acceso al archivo requiere seguir el enlace, lo que implica una operación adicional de lectura del directorio. Si el archivo destino se elimina, el symlink queda apuntando a una ruta inexistente (se convierte en un *dangling link*).

## Indexación, caché y optimización del acceso

Los sistemas operativos utilizan múltiples estrategias para reducir la latencia de acceso a archivos:

**Caché de páginas (*page cache*):** Linux utiliza la memoria libre del sistema como caché para datos leídos de archivos. Los datos que se leen frecuentemente permanecen en memoria, eliminando la necesidad de acceder al disco en cada operación. El page cache puede consumir la mayoría de la RAM disponible, mejorando significativamente el rendimiento de E/S.

**Caché de metadatos (*inode cache*):** los metadatos de los archivos (inodos, estructuras de directorio) se mantienen en memoria para acelerar las operaciones de listado y búsqueda de archivos.

**Caché de directorios (*dentry cache*):** las traducciones de nombres de archivo a inodos se almacenan en una caché para evitar búsquedas repetitivas en el disco.

**Defragmentación:** en discos duros, la fragmentación (archivos divididos en múltiples bloques no contiguos) degrada el rendimiento de lectura secuencial. La defragmentación reorganiza los bloques para que los archivos estén contiguos. Los SSD no requieren defragmentación, ya que no tienen penalización por accesos aleatorios.

> Los SSD han cambiado significativamente las optimizaciones de archivos. Las estrategias que mejoran el rendimiento en HDD (contigüidad de bloques, minimización de seeks) son menos relevantes en SSD, donde la latencia de acceso es prácticamente independiente de la posición física de los datos. Los sistemas de archivos modernos para SSD utilizan estrategias diferentes que priorizan la reducción de escrituras para preservar la vida útil del dispositivo.

