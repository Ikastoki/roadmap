# Fundamentos de la información

La información es el recurso más valioso de un sistema informático. Sin datos que procesar, almacenar y transmitir, un ordenador no tiene propósito. La gestión eficaz de la información —cómo se organiza, se accede, se protege y se mantiene— es una de las disciplinas fundamentales de la informática que afecta directamente la productividad, la seguridad y la fiabilidad de cualquier sistema.

Comprender los fundamentos de la información en un contexto digital implica entender qué es un dato, cómo se representa, cómo se almacena y cómo se recupera. Estos conceptos son la base sobre la cual se construyen los sistemas de archivos, las bases de datos, las redes y prácticamente todos los servicios informáticos que utilizamos diariamente.

## Qué es la información para un sistema digital

Para un sistema informático, la información es una secuencia de bits que representa datos con significado. Un archivo de texto es una secuencia de bytes que representan caracteres; una imagen es una secuencia de bytes que representan píxeles; un vídeo es una secuencia de bytes que representan fotogramas y audio. La diferencia entre un archivo de texto y un archivo de vídeo no está en los bits mismos, sino en la interpretación que el software hace de ellos.

Esta distinción entre datos brutos e información con significado es fundamental. Un byte con el valor `0x41` es solo un número entre 0 y 255 para el hardware, pero el software lo interpreta como la letra "A" en ASCII. El mismo byte `0x41` podría interpretarse como un valor numérico, un color, un comando de impresora o una instrucción, dependiendo del contexto. La semántica de los datos —el significado que les asignamos— reside en el software, no en el hardware.

## El archivo como unidad básica de almacenamiento

Un archivo es la unidad básica de almacenamiento en un sistema de archivos. Representa una secuencia de bytes que se almacena de forma persistente en un medio de almacenamiento (disco duro, SSD, memoria no volátil) y que puede ser leída, modificada o eliminada por el usuario o por programas.

Los archivos son la abstracción fundamental que permite a los programas y a los usuarios almacenar y recuperar información sin preocuparse por los detalles físicos del medio de almacenamiento. Cuando un usuario guarda un documento de texto, el sistema de archivos traduce los bytes del documento en patrones magnéticos o eléctricos en el disco, una operación que el usuario no necesita ni conocer ni comprender.

## Estructura de un archivo

Todo archivo se compone de dos partes fundamentales:

**Metadatos:** información sobre el archivo que no es el contenido del archivo en sí, sino datos descriptivos que el sistema de archivos utiliza para gestionarlo. Los metadatos típicos incluyen:

- **Nombre del archivo:** la designación legible por humanos que identifica al archivo.
- **Tamaño:** la cantidad de bytes que contiene el archivo.
- **Fecha de creación y modificación:** marcas de tiempo que indican cuándo se creó y modificó por última vez el archivo.
- **Permisos:** qué usuarios y grupos pueden leer, escribir o ejecutar el archivo.
- **Propietario:** el usuario y grupo que poseen el archivo.
- **Ubicación física:** las direcciones en el medio de almacenamiento donde se encuentran los bloques de datos del archivo.
- **Tipo de archivo:** una indicación del formato del archivo (texto, imagen, ejecutable, etc.).

**Datos:** el contenido propiamente dicho del archivo, la secuencia de bytes que constituye la información que el usuario o el programa almacenan.

> La separación entre metadatos y datos es una de las decisiones arquitectónicas más importantes de un sistema de archivos. Los sistemas Unix (ext4, ZFS) almacenan los metadatos en estructuras llamadas inodos, mientras que NTFS utiliza una tabla de atributos del archivo ($MFT). Esta decisión afecta al rendimiento, la seguridad y la capacidad de recuperación del sistema.

## El concepto de ruta (*path*) y nombre de archivo

Una ruta (*path*) es la secuencia de directorios y subdirectorios que identifica la ubicación de un archivo dentro de la jerarquía del sistema de archivos. Las rutas permiten localizar y acceder a cualquier archivo del sistema de forma unívoca.

**Ruta absoluta:** comienza desde la raíz del sistema de archivos y especifica la ubicación completa del archivo. En sistemas Unix, la raíz se representa con `/`:

```
/home/usuario/documentos/reporte.pdf
```

**Ruta relativa:** especifica la ubicación del archivo respecto al directorio de trabajo actual. Si el directorio de trabajo es `/home/usuario/`, la ruta relativa al archivo anterior sería:

```
documentos/reporte.pdf
```

Los nombres de archivo modernos soportan caracteres alfanuméricos, guiones, puntos y otros símbolos, aunque existen restricciones según el sistema de archivos. El punto (`.`) separa el nombre del archivo de su extensión, que típicamente indica el tipo de archivo (`.txt`, `.jpg`, `.pdf`). Los sistemas operativos utilizan la extensión para determinar qué aplicación abrir el archivo de forma predeterminada.

