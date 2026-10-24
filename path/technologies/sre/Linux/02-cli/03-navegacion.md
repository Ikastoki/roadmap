# Navegacion por el sistema de archivos

El sistema de archivos de Linux es una estructura jerarquica que comienza en el directorio raiz `/`. Navegar entre directorios y conocer en que punto del arbol nos encontramos es la operacion mas basica y frecuente en la CLI. Sin una comprension solida de las rutas y los comandos de navegacion, el resto de operaciones con archivos carecen de contexto.

## pwd: Print Working Directory

Cada proceso en Linux tiene un directorio de trabajo actual (current working directory). Cuando se ejecuta un comando que opera sobre rutas relativas, estas se interpretan a partir de ese directorio. El comando `pwd` (print working directory) muestra la ruta absoluta del directorio actual.

```bash
$ pwd
/home/usuario/documentos
```

La importancia de `pwd` radica en que la shell no muestra permanentemente la ruta completa en el prompt de许多 configuraciones. Al ejecutar `pwd` se obtiene la confirmacion exacta de la ubicacion actual, especialmente util en scripts o cuando se trabaja con multiples terminales.

Internamente, el kernel mantiene el directorio actual en el descriptor de cada proceso. La shell almacena este valor en la variable `PWD` y el comando `pwd` simplemente la muestra (en Bash, es un builtin). Existe una variante `pwd -P` que muestra la ruta fisica sin seguir enlaces simbolicos, y `pwd -L` que respeta los enlaces (comportamiento por defecto).

```bash
# Ruta fisica (sin enlaces simbolicos)
$ pwd -P
/home/usuario/documentos

# Ruta logica (con enlaces simbolicos)
$ pwd -L
/home/usuario/mis_docs   # si /home/usuario/mis_docs es un enlace a documentos
```

## ls: List directory contents

`ls` enumera el contenido de un directorio. Sin argumentos, muestra los archivos y directorios del directorio actual. Es probablemente el comando mas usado en Linux, y sus opciones determinan la cantidad de informacion que se obtiene de cada elemento.

```bash
# Listado basico
$ ls
Documentos  Descargas  imagenes  script.sh

# Listado detallado con -l
$ ls -l
-rw-r--r-- 1 usuario usuario  1024 ene 15 10:30 Documentos
drwxr-xr-x 2 usuario usuario  4096 ene 14 08:00 Descargas
-rwxr-xr-x 1 usuario usuario   256 ene 13 22:15 script.sh
```

El formato largo (`-l`) muestra por cada entrada: tipo de archivo y permisos, numero de enlaces, propietario, grupo, tamano, fecha de modificacion y nombre.

La opcion `-a` (all) incluye los archivos ocultos, aquellos cuyo nombre comienza por un punto (`.`). Cada directorio contiene dos entradas especiales: `.` (el propio directorio) y `..` (el directorio padre).

```bash
$ ls -a
.  ..  .bashrc  .config  Documentos  .gitconfig  .local
```

La opcion `-h` (human-readable) combina con `-l` para mostrar los tamanos en formato legible (KB, MB, GB) en lugar de bytes.

```bash
$ ls -lh
-rw-r--r-- 1 usuario usuario 1.2K ene 15 10:30 archivo.txt
-rw-r--r-- 1 usuario usuario  45M ene 14 08:00 video.mp4
```

Estas opciones se combinan habitualmente en `ls -lah`, que muestra todos los archivos (incluidos ocultos) con detalle y tamanos legibles. Es el equivalente al "ver archivos ocultos y detalles" de un explorador grafico.

## Rutas relativas y absolutas

Una ruta es la direccion que localiza un archivo o directorio dentro del arbol del sistema de archivos. Linux distingue dos tipos fundamentales:

Las **rutas absolutas** comienzan con `/` y describen la ubicacion completa desde la raiz del sistema hasta el archivo. Son inequivocas: `/home/usuario/documentos/informe.pdf` siempre se refiere al mismo archivo, independientemente del directorio de trabajo actual.

```bash
# Ejemplos de rutas absolutas
/etc/ssh/sshd_config
/var/log/syslog
/home/usuario/.bashrc
```

Las **rutas relativas** se interpretan a partir del directorio de trabajo actual. No comienzan con `/`. Utilizan dos referencias especiales:

- `.` (un punto): el directorio actual.
- `..` (dos puntos): el directorio padre del actual.

```bash
# Suponiendo que el directorio actual es /home/usuario

# Acceder a un subdirectorio
cd Documentos          # Equivalente a: cd /home/usuario/Documentos

# Subir un nivel
cd ..                  # Ahora en /home

# Subir dos niveles
cd ../..               # Ahora en /

# Ruta relativa desde /home/usuario a /etc
ls ../../etc/ssh       # Lista /etc/ssh desde cualquier lugar
```

El arbol de directorios puede visualizarse como:

```
/
├── etc/
│   └── ssh/
│       └── sshd_config
├── home/
│   └── usuario/
│       ├── Documentos/
│       └── .bashrc
└── var/
    └── log/
        └── syslog
```

Si el directorio actual es `/home/usuario/Documentos`, la ruta relativa `../../etc/ssh/sshd_config` equivale a la absoluta `/etc/ssh/sshd_config`. Las rutas relativas hacen que los comandos sean portables entre sistemas si la estructura relativa se mantiene, pero pueden causar errores si no se conoce con exactitud el directorio actual.

> Un error frecuente es asumir que un script se ejecuta desde el directorio donde reside. El directorio actual es el del proceso que invoca el script, no el del propio script. Por eso los scripts robustos suelen comenzar con `cd "$(dirname "$0")"` para situarse en su propio directorio antes de usar rutas relativas.

