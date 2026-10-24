# Variables

Las variables en Bash son contenedores que almacenan datos, casi siempre como cadenas de texto, que pueden ser referenciados y modificados durante la ejecucion del script. A diferencia de los lenguajes compilados, Bash no requiere declaracion de tipo: toda variable es texto hasta que el contexto de uso le da un significado diferente.

## Variables locales

Se crean asignando un valor con el operador `=`. No debe haber espacios alrededor del signo igual. El nombre de variable puede contener letras, numeros y guiones bajos, pero no puede comenzar con un numero.

```bash
nombre="Juan"
edad=25
no_espacios="esto funciona"
```

Para acceder al valor se antepone el simbolo `$` al nombre. Las llaves `{}` son opcionales pero recomendadas cuando la variable aparece junto a otros caracteres sin espacio:

```bash
echo $nombre
echo "Me llamo ${nombre} y tengo ${edad} anos"
```

## Variables de entorno

Las variables de entorno son heredadas por los procesos hijos. Se definen con `export` y suelen almacenar configuracion del sistema o del usuario:

```bash
export PATH="$PATH:/ruta/personal"
export EDITOR=vim
```

El comando `env` muestra todas las variables de entorno activas. Algunas de las mas importantes son `HOME` (directorio personal), `PATH` (rutas de busqueda de ejecutables), `USER` (nombre del usuario actual), `SHELL` (ruta del shell en uso) y `PWD` (directorio actual de trabajo).

## Convenciones de nombres

- **Mayusculas** para variables de entorno y constantes: `PATH`, `MAX_LIMIT=100`.
- **Minusculas** o **snake_case** para variables locales: `contador`, `archivo_salida`.
- Sin espacios alrededor de `=`.
- Nombres descriptivos y en ingles o espanol, consistentemente.

## Expansion de variables

Bash realiza la expansion de variables antes de ejecutar el comando. Esto permite sustituir valores en cualquier parte del comando:

```bash
DIR="/etc"
ls $DIR            # expande a ls /etc
ARCHIVO="log.txt"
cat "${DIR}/${ARCHIVO}"   # expande a cat /etc/log.txt
```

La expansion con `{}` tambien permite modificaciones sobre el valor:

```bash
v="hola_mundo"
echo ${v^^}          # HOLA_MUNDO (mayusculas)
echo ${v,,}          # HOLA_MUNDO (minusculas)
echo ${v#hola_}      # mundo (elimina prefijo)
echo ${v%_mundo}     # hola (elimina sufijo)
echo ${v/_/ }        # hola mundo (sustituye patron)
```

## Variables especiales

Bash proporciona variables predefinidas que proporcionan informacion sobre el estado del script y los argumentos recibidos:

| Variable | Descripcion |
|----------|-------------|
| `$?` | Codigo de salida del ultimo comando ejecutado (0 = exito) |
| `$#` | Numero de argumentos pasados al script |
| `$@` | Todos los argumentos como una lista (cada uno entrecomillado por separado) |
| `$*` | Todos los argumentos como una unica cadena |
| `$0` | Nombre del script |
| `$1` a `$9` | Argumentos posicionales 1 a 9 |
| `$$` | PID del proceso actual del script |
| `$!` | PID del ultimo proceso ejecutado en segundo plano |
| `$LINENO` | Numero de linea actual en el script |

```bash
#!/bin/bash
echo "Script: $0"
echo "Argumentos: $#"
echo "Primer arg: $1"
echo "Todos: $@"
ls /no/existe 2>/dev/null
echo "Codigo salida: $?"   # muestra 1 o 2 (comando fallo)
```

> `$?` se resetea con cada comando ejecutado. Si se necesita consultar el codigo de salida mas adelante, debe guardarse en una variable inmediatamente despues del comando cuyo resultado interesa.

```bash
comando_importante
CODIGO=$?
# ... mas codigo ...
if [ $CODIGO -eq 0 ]; then
    echo "Comando exitoso"
fi
```

