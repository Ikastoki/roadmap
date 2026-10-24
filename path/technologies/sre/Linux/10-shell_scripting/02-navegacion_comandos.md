# Navegacion y manipulacion del sistema

El sistema de archivos de Linux es una estructura jerarquica en forma de arbol invertido cuya raiz es `/`. Cada directorio y archivo ocupa una posicion dentro de este arbol, accesible mediante rutas. Comprender las rutas, los permisos y los comandos esenciales de manipulacion es requisito indispensable para trabajar con el shell de forma efectiva.

## Rutas absolutas y relativas

Una **ruta absoluta** comienza desde la raiz del sistema de archivos y describe la ubicacion completa de un elemento. Por ejemplo, `/home/usuario/documentos/informe.txt` localiza el archivo sin ambiguedad, independientemente del directorio actual de trabajo. Una **ruta relativa**, en cambio, se expresa en funcion del directorio actual. Si se esta en `/home/usuario`, la ruta `documentos/informe.txt` es relativa y equivale a la absoluta anterior.

Dos entradas especiales hacen posible la navegacion relativa: `.` representa el directorio actual y `..` el directorio padre. Asi, `cd ..` asciende un nivel en la jerarquia, y `ls ../../etc` lista el contenido del directorio `/etc` desde cualquier punto cuyo abuelo sea la raiz.

```bash
pwd                      # muestra /home/usuario
cd /var/log              # ruta absoluta
ls ../../etc             # ruta relativa hacia /etc
cd -                     # vuelve al directorio anterior
```

## Gestion de archivos y directorios

Los comandos fundamentales para manipular archivos y directorios forman el vocabulario minimo de cualquier usuario del shell:

- `ls` lista el contenido de un directorio. Con `-l` muestra permisos, propietario, tamano y fecha; con `-a` incluye archivos ocultos; con `-h` humaniza los tamanos.
- `mkdir` crea directorios. La opcion `-p` crea directorios padres intermedios si no existen.
- `cp` copia archivos o directorios (con `-r` para copia recursiva).
- `mv` mueve o renombra archivos y directorios.
- `rm` elimina archivos. `rm -r` elimina directorios recursivamente; `rm -f` fuerza la eliminacion sin confirmacion.

```bash
mkdir -p proyectos/web/css
cp -r /etc/ssh ~/backups/ssh_backup
mv informe_v1.txt informe_final.txt
rm -rf /tmp/cache_temp
```

> La opcion `-f` de `rm` suprime las confirmaciones, lo que la hace peligrosa en scripts automatizados si no se valida la ruta antes de ejecutar la eliminacion.

## Permisos y propietarios

Cada archivo y directorio en Linux tiene un propietario (usuario), un grupo y tres conjuntos de permisos: lectura (`r`), escritura (`w`) y ejecucion (`x`). Estos tres conjuntos se aplican al propietario (u), al grupo (g) y a otros usuarios (o). Los permisos se representan en `ls -l` como una cadena de 10 caracteres: el primero indica el tipo de archivo (`-` regular, `d` directorio, `l` enlace simbolico), y los nueve siguientes son los permisos en grupos de tres.

```bash
-rwxr-xr--  1 root root  1234 Jul 28 10:00 script.sh
# |  |  |            permisos: u=rwx, g=r-x, o=r--
# |  |  +-- otros (o)
# |  +----- grupo (g)
# +-------- usuario (u)
```

`chmod` modifica los permisos. Acepta notacion octal (cada permiso tiene un valor numerico: `r=4`, `w=2`, `x=1`) o notacion simbolica (`u+x` anade ejecucion para el usuario, `o-r` quita lectura para otros).

```bash
chmod 755 script.sh       # rwxr-xr-x
chmod u+x script.sh       # anade ejecucion al usuario
chmod -R 644 docs/        # cambia permisos recursivamente
chown usuario:grupo archivo
chgrp grupo archivo
```

`chown` cambia el propietario y grupo de un archivo. `chgrp` cambia solo el grupo. Ambas operaciones requieren privilegios de superusuario a menos que el usuario sea el propietario del archivo y solo desee cambiar el grupo a uno del que forme parte.

## Comodines (glob patterns)

El shell interpreta patrones de busqueda llamados globs antes de ejecutar los comandos:

- `*` coincide con cualquier secuencia de caracteres (incluyendo cero).
- `?` coincide con un unico caracter.
- `[abc]` coincide con cualquier caracter dentro del conjunto.
- `[a-z]` coincide con cualquier caracter en el rango.

```bash
ls *.txt                  # todos los .txt
rm foto?.jpg              # foto1.jpg, foto2.jpg, no foto10.jpg
cp [aeiou]* /tmp/         # archivos que empiezan por vocal
```

