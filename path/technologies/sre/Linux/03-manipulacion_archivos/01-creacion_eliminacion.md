# Creacion y eliminacion de archivos y directorios

El sistema de archivos de Linux no es una abstraccion pasiva: cada archivo o directorio que existe representa una entrada en una tabla de inodos, y cada operacion de creacion o eliminacion implica la manipulacion directa de esa tabla. Crear un archivo vacio no solo reserva un nombre en el directorio padre, sino que asigna un inodo numero unico que quedara registrado hasta que el ultimo enlace duro que apunte a el sea eliminado. Comprender que hay detras de `mkdir`, `touch`, `rmdir` y `rm` permite anticipar el comportamiento del sistema cuando los limites de inodos se agotan, cuando un archivo ocupado no puede eliminarse o cuando la recuperacion de datos se vuelve imposible.

## Creacion de directorios con mkdir

El comando `mkdir` (del ingles _make directory_) crea un directorio nuevo en la ubicacion especificada. Un directorio en Linux no es mas que un archivo especial que contiene una tabla de asociaciones entre nombres y numeros de inodo. Cuando se ejecuta `mkdir`, el kernel reserva un inodo para el nuevo directorio, lo inicializa con las entradas estandar `.` (referencia a si mismo) y `..` (referencia al directorio padre), y anade una entrada en el directorio padre que asocia el nombre proporcionado con ese nuevo inodo.

```bash
mkdir directorio
```

El comportamiento por defecto de `mkdir` es rechazar la operacion si el directorio padre no existe. Esto evita la creacion accidental de arboles de directorios incompletos que dejarian el sistema de archivos en un estado inconsistente desde la perspectiva del usuario.

```bash
mkdir nivel1/nivel2/nivel3
# Error: mkdir: cannot create directory 'nivel1/nivel2/nivel3': No such file or directory
```

La opcion `-p` (_parents_) elimina esta restriccion y crea toda la cadena de directorios intermedios que falten:

```bash
mkdir -p nivel1/nivel2/nivel3
# Crea nivel1, nivel1/nivel2 y nivel1/nivel2/nivel3
```

Esta opcion no produce error si los directorios ya existen, lo que la convierte en una herramienta segura para scripts que necesitan garantizar la existencia de una estructura de directorios antes de proceder. Sin `-p`, el script fallaria al segundo intento.

> `mkdir -p` es idiomatico en scripts de instalacion y despliegue. Es comun verlo combinado con `&&` para encadenar operaciones que dependen de la existencia del directorio: `mkdir -p build/logs && cp output.log build/logs/`.

Otros parametros relevantes incluyen `-v` (_verbose_) para que muestre cada directorio creado, y `-m` (_mode_) para establecer permisos directamente en el momento de la creacion, evitando una llamada posterior a `chmod`:

```bash
mkdir -m 700 directorio_privado
```

## Creacion de archivos con touch

`touch` es, en apariencia, el comando mas sencillo del sistema: recibe una ruta y, si el archivo no existe, lo crea vacio. Sin embargo, su proposito original no es crear archivos, sino actualizar las marcas temporales (_timestamps_) de un archivo existente: el tiempo de acceso (`atime`) y el tiempo de modificacion (`mtime`). La creacion de archivos vacios es un efecto colateral de este comportamiento.

```bash
touch archivo_nuevo.txt
ls -l archivo_nuevo.txt
# -rw-rw-r-- 1 user user 0 Jul 28 10:00 archivo_nuevo.txt
```

Cuando `touch` opera sobre un archivo que ya existe, actualiza sus timestamps a la hora actual sin modificar el contenido:

```bash
touch archivo_existente.txt
```

El comando acepta opciones para manipular los timestamps de forma precisa. La opcion `-t` permite establecer una fecha y hora especifica en formato `[[CC]YY]MMDDhhmm[.ss]`:

```bash
touch -t 202501011200.00 archivo_con_fecha_antigua.txt
```

> `touch` es util en escenarios de compilacion condicional: al actualizar el timestamp de un archivo fuente con `touch`, herramientas como `make` interpretan que el archivo ha sido modificado y recompilan los objetivos que dependen de el.

Opciones adicionales incluyen `-a` para modificar solo el `atime`, `-m` para modificar solo el `mtime`, y `-c` para evitar la creacion del archivo si no existe (actualiza timestamps solo sobre archivos existentes).

El archivo creado por `touch` ocupa cero bloques de datos en el disco. Sin embargo, consume un inodo, que es un recurso limitado en sistemas de archivos tradicionales como ext4. En sistemas con millones de archivos diminutos, el limite de inodos puede alcanzarse antes que el limite de espacio en disco, impidiendo la creacion de nuevos archivos aunque haya espacio disponible.

## Eliminacion de directorios con rmdir

`rmdir` (_remove directory_) elimina un directorio, pero unicamente si esta vacio. Esta restriccion no es un capricho de diseno: responde a la necesidad de evitar la eliminacion accidental de arboles completos de archivos.

```bash
rmdir directorio_vacio
# El directorio se elimina si no contiene archivos ni subdirectorios (excepto . y ..)
```

Si el directorio contiene cualquier archivo o subdirectorio, `rmdir` falla con un mensaje de error:

```bash
rmdir directorio_con_contenido
# Error: rmdir: failed to remove 'directorio_con_contenido': Directory not empty
```

Al igual que `mkdir`, `rmdir` acepta la opcion `-p` para eliminar directorios anidados de forma recursiva, siempre que todos ellos esten vacios:

```bash
rmdir -p nivel1/nivel2/nivel3
# Elimina nivel3, luego nivel2 (si quedo vacio), luego nivel1 (si quedo vacio)
```

`rmdir` es conservador por diseno. En la practica, su uso se limita a scripts que necesitan una eliminacion segura de directorios temporales o a operaciones donde se quiere verificar explicitamente que un directorio esta vacio antes de eliminarlo.

## Eliminacion recursiva con rm

`rm` (_remove_) es el comando general para eliminar archivos y, con las opciones adecuadas, directorios. A diferencia de `rmdir`, `rm` no impone restricciones de contenido: con los permisos suficientes, elimina cualquier entrada del sistema de archivos.

```bash
rm archivo.txt
# Elimina el archivo de forma permanente
```

Para eliminar un directorio y todo su contenido, es necesaria la opcion `-r` (_recursive_):

```bash
rm -r directorio/
# Elimina el directorio, todos sus archivos y todos sus subdirectorios
```

La recursividad implica que `rm` atraviesa el arbol de directorios en profundidad, eliminando cada archivo y cada subdirectorio antes de eliminar el directorio raiz. No existe papelera de reciclaje: una vez que `rm` confirma la operacion, los datos no son recuperables mediante comandos estandar del sistema operativo.

La opcion `-f` (_force_) elimina dos barreras de seguridad. Primera: suprime cualquier pregunta de confirmacion, incluso si el archivo tiene permisos de solo lectura. Segunda: no muestra errores si el archivo o directorio no existe.

```bash
rm -rf directorio/
# Elimina sin preguntar, incluso si existen archivos protegidos contra escritura
```

`rm -f` no elimina archivos cuyo directorio padre tenga permisos de escritura denegados para el usuario, ya que la eliminacion es una operacion que modifica el directorio, no el archivo en si. Sin embargo, si el archivo tiene permisos de solo lectura pero el directorio es escribible, `rm` preguntara confirmacion a menos que se use `-f`.

> La combinacion `rm -rf /` es la forma mas conocida de causar danos irreparables en un sistema Linux. Las versiones modernas de `rm` incluyen una salvaguarda que detecta cuando se intenta eliminar la raiz del sistema de archivos y rechaza la operacion a menos que se anada `--no-preserve-root`. No obstante, `rm -rf .` ejecutado desde el directorio raiz logra el mismo efecto eludiendo esa proteccion.

## Riesgos de rm -f y buenas practicas

La opcion `-f` elimina la confirmacion incluso cuando se combina con patrones comodin. Un error tipografico en un comodín puede tener consecuencias devastadoras:

```bash
# Intencion: eliminar archivos .tmp
rm -f *.tmp

# Error tipografico: elimina todo
rm -f * .tmp   # El espacio entre * y .tmp hace que * expanda a todos los archivos
```

Las repercusiones de `rm -rf` mal empleado incluyen:

- **Perdida total de datos**: no existe comando `undo` o `restore` nativo en Linux. Las herramientas de recuperacion de archivos (como `testdisk` o `extundelete`) solo funcionan si los bloques de datos no han sido sobrescritos, lo que ocurre rapidamente en sistemas en uso.
- **Dano al sistema operativo**: eliminar directorios del sistema como `/lib`, `/etc` o `/usr` deja la distribucion inutilizable, requiriendo reinstalacion completa.
- **Propagacion en montajes**: si existen sistemas de archivos montados dentro del arbol que se elimina, `rm -rf` los atraviesa y elimina su contenido. La opcion `--one-file-system` evita este comportamiento al limitar la operacion al sistema de archivos actual:

```bash
rm -rf --one-file-system /directorio/con_montajes/
```

Las buenas practicas para mitigar estos riesgos incluyen:

1. Usar `-i` (_interactive_) para que `rm` pregunte antes de cada eliminacion: `rm -ri directorio/`.
2. Verificar siempre el resultado de la expansion de comodines con `echo` antes de ejecutar `rm`: `echo *.tmp` confirma que los archivos que se van a eliminar son los esperados.
3. Establecer alias protectores en el archivo de configuracion de la shell, como `alias rm='rm -i'`, aunque esto no protege contra `rm -f` (que sobreescribe `-i`).
4. Utilizar `trash-cli` como alternativa: `trash-put archivo` mueve el archivo a una papelera desde la que puede recuperarse, emulando el comportamiento de los entornos graficos.

