# Estructura de los comandos en Linux

Cada instruccion que se introduce en la shell sigue una estructura formal que el interprete descompone para determinar que programa ejecutar y con que parametros. Comprender esta estructura es esencial no solo para usar los comandos correctamente, sino para leer documentacion, diagnosticar errores y escribir scripts predecibles.

Un comando en Linux no es mas que un programa —compilado o interpretado— al que la shell localiza, ejecuta y le pasa los argumentos que el usuario escribe. La shell no entiende el significado de las opciones; simplemente las transmite al programa, que es quien decide como interpretarlas.

## Estructura basica

La sintaxis general de cualquier comando en Linux sigue este patron:

```
comando [opciones] [argumentos]
```

- **comando**: el nombre del programa a ejecutar. Puede ser un binario en el sistema (`/bin/ls`), un comando interno de la shell (`cd`) o un alias definido por el usuario.
- **opciones**: modifican el comportamiento del comando. Suelen precederse con un guion. Permiten activar o desactivar funcionalidades especificas.
- **argumentos**: los objetos sobre los que actua el comando. Normalmente son archivos, directorios, cadenas de texto o numeros.

```bash
# Ejemplo: comando con opciones y argumentos
ls -l --color=auto /home/usuario
#  ^    ^             ^
#  |    |             argumento: directorio
#  |    opcion larga con valor
#  opcion corta
```

Los corchetes en la notacion documental indican que el elemento es opcional. No todos los comandos requieren argumentos: `pwd` no necesita ninguno, mientras que `cp` exige al menos dos (origen y destino). La posicion relativa de opciones y argumentos puede variar segun el programa, aunque la convencion POSIX recomienda escribir las opciones antes que los argumentos.

## Comandos internos y externos

Una distincion fundamental en la shell es si un comando es interno (builtin) o externo (un binario en el sistema de archivos).

Los **comandos internos** estan incorporados directamente en el codigo de la shell. No existe un binario independiente para ellos; cuando se ejecutan, la shell no crea un proceso hijo, sino que los resuelve internamente. Ejemplos tipicos son `cd`, `echo`, `export`, `alias`, `source` y `exit`. Al ser parte de la shell, se ejecutan sin la sobrecarga de buscar un binario en el PATH, lanzar un proceso hijo y esperar su finalizacion. Tambien pueden modificar el estado interno de la shell, algo que un proceso externo no podria hacer (como cambiar el directorio de trabajo con `cd`).

Los **comandos externos** son programas independientes almacenados en el sistema de archivos, normalmente en directorios como `/bin`, `/usr/bin` o `/usr/local/bin`. Cuando se ejecuta un comando externo, la shell busca el binario en los directorios listados en la variable de entorno `PATH`, lo ejecuta como un proceso hijo y espera a que termine. Ejemplos son `ls`, `grep`, `awk`, `python` y `nginx`.

```bash
# Verificar si un comando es interno o externo
type cd      # cd is a shell builtin
type ls      # ls is /usr/bin/ls
type -a echo # Muestra todas las coincidencias (builtin y externo)
```

> Algunos comandos existen tanto como internos de la shell como binarios externos. Por ejemplo, `echo` es un builtin en Bash y Zsh, pero tambien existe `/bin/echo`. La shell ejecuta primero el builtin. El comando `type` revela cual se ejecutara realmente.

## Opciones cortas y opciones largas

Las opciones de un comando pueden expresarse en dos formatos:

Las **opciones cortas** consisten en un guion seguido de una unica letra. Varias opciones cortas pueden agruparse detras de un solo guion:

```bash
# Opciones separadas
ls -l -a -h

# Opciones agrupadas (equivalente a la anterior)
ls -lah
```

Las **opciones largas** usan dos guiones seguidos de una palabra descriptiva. Son mas legibles y faciles de recordar, sobre todo para opciones poco frecuentes. No pueden agruparse:

```bash
# Equivalente con opciones largas
ls --format=long --all --human-readable
```

Algunas opciones largas requieren un valor asociado, que puede separarse con un espacio o con el signo `=`:

```bash
# Opcion larga con valor
tar --file archivo.tar --create --verbose

# Equivalente con el signo =
tar --file=archivo.tar --create --verbose
```

En las opciones cortas, el valor suele ir separado por un espacio, aunque en algunos comandos puede ir sin el:

```bash
# Opcion corta con valor
sort -t , -k 2 archivo.csv

# En algunos comandos, el valor puede ir pegado
sort -t, -k2 archivo.csv
```

> La convencion POSIX establece que las opciones cortas deben ser una unica letra y precederse de un guion. Las opciones largas son una extension de GNU y no forman parte del estandar POSIX original, pero estan implementadas en la mayoria de herramientas GNU/Linux. Cuando se escribe documentacion portable, conviene usar opciones cortas; para scripts personales o internos, las opciones largas mejoran la legibilidad.

```bash
# Ejemplo comparativo con el mismo comando
# Estilo corto (portable, conciso)
ps aux

# Estilo largo (legible, especifico de GNU)
ps --user=root --format=user,pid,%cpu,%mem
```

