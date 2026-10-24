# Copia, movimiento y renombrado de archivos

La distincion entre copiar y mover un archivo refleja una diferencia fundamental en como el sistema de archivos gestiona los datos y los metadatos. Copiar implica duplicar el contenido en un nuevo inodo, ocupando espacio adicional en disco. Mover, en cambio, es una operacion sobre el directorio: el contenido permanece intacto en el mismo inodo, y solo se actualiza la entrada en la tabla del directorio. Comprender esta diferencia es clave para optimizar el uso del disco, anticipar el comportamiento ante enlaces duros y entender por que mover un archivo entre sistemas de archivos distintos puede ser mucho mas lento que dentro del mismo.

## Copia de archivos con cp

`cp` (_copy_) crea un duplicado exacto de un archivo. El kernel lee el contenido del archivo origen, reserva un nuevo inodo en el destino y escribe los datos en los bloques asignados. El resultado son dos archivos independientes: modificar uno no afecta al otro.

```bash
cp origen.txt destino.txt
```

Si el archivo destino ya existe, `cp` lo sobrescribe sin previo aviso. Para solicitar confirmacion antes de sobrescribir, se utiliza la opcion `-i` (_interactive_):

```bash
cp -i origen.txt destino.txt
# Pregunta antes de sobrescribir
```

La opcion `-v` (_verbose_) muestra cada archivo que se esta copiando, util para verificar el progreso en operaciones con multiples archivos:

```bash
cp -v *.txt copias/
# 'archivo1.txt' -> 'copias/archivo1.txt'
# 'archivo2.txt' -> 'copias/archivo2.txt'
```

Para preservar los metadatos del archivo original (permisos, timestamps, propietario), se utiliza `-p` (_preserve_):

```bash
cp -p origen.txt destino.txt
# Copia contenido y preserva permisos, timestamps y propiedad
```

Sin `-p`, el archivo copiado hereda los permisos determinados por la mascara de usuario (`umask`) y la hora actual como timestamp. La opcion `-a` (_archive_) combina `-p` con `-d` (preservar enlaces simbolicos) y `-r` (recursividad), siendo equivalente a `cp -dR --preserve=all`.

> La copia de un archivo no preserva los enlaces duros que apuntan a el. Si dos nombres de archivo apuntan al mismo inodo mediante enlaces duros, copiar uno de ellos produce un nuevo archivo independiente que ocupa espacio adicional. La opcion `--preserve=links` intenta mantener esta relacion, pero tiene limitaciones.

## Copia de directorios con cp -r

Los directorios requieren la opcion `-r` (_recursive_) para copiarse. Sin ella, `cp` rechaza la operacion:

```bash
cp directorio/ destino/
# Error: omitting directory 'directorio/'
```

Con `-r`, `cp` recorre el arbol de directorios de forma recursiva, creando los directorios necesarios en el destino y copiando cada archivo:

```bash
cp -r proyecto/ backup_proyecto/
```

La opcion `-u` (_update_) solo copia archivos cuyo timestamp de modificacion sea mas reciente que el del destino, o que no existan en el destino. Es util para sincronizaciones incrementales sin herramientas externas como `rsync`:

```bash
cp -ru origen/ destino/
```

El comportamiento de `cp -r` respecto a los enlaces simbolicos dentro del arbol se controla con las opciones `-P` (preservar el enlace, comportamiento por defecto) y `-L` (seguir el enlace y copiar el archivo apuntado). La diferencia es sutil pero importante: `-P` copia el enlace como enlace; `-L` copia el contenido del archivo destino del enlace, rompiendo la referencia indirecta.

## Movimiento y renombrado con mv

`mv` (_move_) traslada un archivo o directorio de una ubicacion a otra. Cuando el origen y el destino residen en el mismo sistema de archivos, la operacion es casi instantanea: el kernel solo modifica la entrada del directorio, actualizando el nombre y la ruta asociada al inodo. El contenido del archivo nunca se mueve en el disco.

```bash
mv archivo.txt nuevo_nombre.txt        # Renombrar en el mismo directorio
mv archivo.txt directorio/              # Mover a otro directorio
mv archivo.txt directorio/nuevo_nombre.txt  # Mover y renombrar simultaneamente
```

Si el destino se encuentra en un sistema de archivos diferente (por ejemplo, de ext4 a NTFS), `mv` no puede limitarse a actualizar la entrada del directorio. El kernel detecta que los dispositivos son distintos y realiza internamente una operacion de copia seguida de eliminacion del origen. En ese caso, el tiempo de ejecucion depende del tamano del archivo y de la velocidad de lectura/escritura de ambos dispositivos.

```bash
# Movimiento entre sistemas de archivos: copia + eliminacion
mv /home/user/archivo_grande.iso /mnt/disco_externo/
```

`mv` acepta `-i` para confirmar antes de sobrescribir un destino existente, y `-u` para mover solo cuando el origen es mas reciente que el destino:

```bash
mv -i origen.txt destino.txt
# Pregunta si destino.txt existe
```

La opcion `-v` muestra cada movimiento que se realiza, y `-n` (_no-clobber_) evita la sobrescritura de archivos existentes sin preguntar:

```bash
mv -n *.log logs/
# No sobrescribe ningun archivo existente en logs/
```

> A diferencia de `cp`, `mv` conserva los enlaces duros. Al mover un archivo dentro del mismo sistema de archivos, el inodo no cambia, por lo que todos los enlaces duros que apuntaban a el siguen siendo validos. Esto no ocurre al cruzar sistemas de archivos, ya que el nuevo inodo en el destino rompe la relacion.

## Renombrado multiple y estrategias

`mv` opera sobre un unico archivo por invocacion. Para renombrar multiples archivos siguiendo un patron, se combina con bucles de la shell o con herramientas como `rename`:

```bash
# Renombrar todos los .txt a .md con un bucle for
for f in *.txt; do
    mv "$f" "${f%.txt}.md"
done
```

El operador `${f%.txt}` realiza una expansion de parametros en la shell que elimina el sufijo `.txt` de la variable `f`, permitiendo anadir el nuevo sufijo `.md`. Esta tecnica es puramente de shell y no requiere comandos externos.

El comando `rename` (disponible en dos variantes, la de Perl y la de util-linux) ofrece una sintaxis mas concisa basada en sustitucion:

```bash
# Variante Perl: rename 's/\.txt$/.md/' *.txt
# Variante util-linux: rename .txt .md *.txt
```

> La distincion entre mover y renombrar es artificial desde la perspectiva del kernel. `mv` siempre realiza la misma operacion: cambia la ruta asociada a un inodo. Si la ruta cambia de nombre pero permanece en el mismo directorio, el efecto es un renombrado. Si cambia de directorio, es un movimiento. La operacion subyacente es identica.

