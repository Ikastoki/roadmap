# Edicion basica y uso de comodines

La edicion de archivos en terminal no es un lujo de epocas pasadas: es una necesidad cuando se trabaja sobre servidores remotos sin interfaz grafica, se depuran configuraciones en contenedores minimos o se automatizan cambios mediante scripts. Dos editores cubren practicamente todos los escenarios: `nano` para intervenciones rapidas e intuitivas, y `vim` para edicion avanzada con un modelo modal que recompensa la practica con velocidad. Paralelamente, los comodines de la shell (`*`, `?`, `[]`) son el lenguaje con el que se expresan patrones sobre nombres de archivos, y su dominio separa a quien escribe comandos de quien realmente los controla.

## Edicion basica con nano

`nano` es un editor de terminal no modal: las teclas que se pulsan insertan texto directamente. Su filosofia es que cualquier persona pueda usarlo sin curva de aprendizaje, lo que lo convierte en la opcion por defecto para ediciones rapidas en sistemas donde esta instalado (la mayoria de las distribuciones lo incluyen por defecto).

```bash
nano archivo.txt
```

La interfaz de `nano` muestra en la parte inferior una barra con los atajos de teclado mas comunes. El acento circunflejo (`^`) representa la tecla `Ctrl`:

- `Ctrl+O`: guardar el archivo (WriteOut). Pregunta el nombre del archivo; se pulsa `Enter` para confirmar.
- `Ctrl+X`: salir de nano. Si hay cambios sin guardar, pregunta si se desea guardar.
- `Ctrl+K`: cortar la linea actual (o las seleccionadas).
- `Ctrl+U`: pegar el texto cortado.
- `Ctrl+W`: buscar texto. Se escribe el patron y se pulsa `Enter`; `Alt+W` repite la busqueda.
- `Ctrl+\\`: buscar y reemplazar.
- `Ctrl+G`: mostrar la ayuda completa.

Navegar por el archivo se hace con las flechas de cursor, `AvPag`/`RePag`, y combinaciones como `Ctrl+A` (inicio de linea) y `Ctrl+E` (fin de linea).

```bash
# Si el archivo no existe, nano lo crea al guardar
nano /etc/nginx/sites-available/misitio
```

nano no requiere saber si el archivo es nuevo: abre el editor con un buffer vacio y el nombre de archivo indicado. Al guardar, escribe el contenido en la ruta especificada, creando el archivo si no existia.

> En sistemas donde nano no esta instalado, `vi` (la version reducida de vim) suele estar presente en su lugar. `vi` es un editor modal, y un usuario no familiarizado puede quedar atrapado sin saber como salir. El comando `:q!` y `Enter` desde el modo normal (pulsando `Esc` primero) cierra el editor sin guardar cambios.

## Introduccion a vim

`vim` (_Vi IMproved_) es la evolucion del editor clasico `vi`. Su caracteristica definitoria es el modelo modal: el comportamiento de las teclas cambia segun el modo activo. La inversion inicial en aprender los modos se amortiza con creces en velocidad de edicion, ya que las operaciones mas frecuentes (navegar, cortar, copiar, pegar) se ejecutan sin mover las manos de la fila central del teclado.

### Modos de operacion

vim tiene tres modos principales:

- **Modo normal**: es el modo por defecto al abrir vim. Cada tecla ejecuta un comando de navegacion o manipulacion. `j` baja una linea, `k` sube una linea, `dd` elimina la linea actual, `yy` copia la linea actual. Para volver al modo normal desde cualquier otro modo, se pulsa `Esc`.

- **Modo insercion**: permite escribir texto en el archivo. Se accede desde el modo normal con teclas como `i` (insertar en la posicion del cursor), `a` (insertar despues del cursor), `o` (abrir nueva linea debajo) u `O` (abrir nueva linea encima). La barra de estado inferior muestra `-- INSERTAR --`.

- **Modo visual**: permite seleccionar texto. Se accede con `v` (seleccion caracter a caracter), `V` (seleccion por lineas) o `Ctrl+v` (seleccion por bloques). Una vez seleccionado, se pueden aplicar operaciones como `d` (eliminar), `y` (copiar) o `c` (cambiar).

```bash
vim archivo.txt
```

### Comandos esenciales del modo normal

| Comando | Accion |
|---------|--------|
| `h`, `j`, `k`, `l` | Navegacion: izquierda, abajo, arriba, derecha |
| `w`, `b` | Saltar palabra siguiente/anterior |
| `0`, `$` | Inicio/fin de linea |
| `gg`, `G` | Inicio/fin del archivo |
| `dd` | Eliminar linea |
| `yy` | Copiar linea |
| `p`, `P` | Pegar debajo/encima del cursor |
| `u` | Deshacer |
| `Ctrl+r` | Rehacer |
| `/patron` | Buscar patron |
| `:w` | Guardar archivo |
| `:q` | Salir de vim (si no hay cambios) |
| `:q!` | Salir sin guardar |
| `:wq` | Guardar y salir |

### Operaciones compuestas

La potencia de vim reside en la combinacion de verbos y objetos. Un comando como `d2w` significa "elimina dos palabras": `d` es el verbo (delete), `2` es el numero de repeticiones, `w` es el objeto (word). Este patron se aplica a multiples verbos (`c` para cambiar, `y` para copiar, `v` para seleccionar) y objetos (`w` palabra, `j` linea, `}` parrafo, `G` final del archivo).

```vim
ci"   " Cambia todo el texto dentro de unas comillas dobles
d}    " Elimina desde el cursor hasta el final del parrafo
y3j   " Copia el cursor y las tres lineas siguientes
```

> Aprender a salir de vim es el primer obstaculo de todo principiante. Si el editor no responde, la secuencia `Esc` seguida de `:q!` y `Enter` cierra la sesion sin guardar. Si se quiere guardar antes de salir, `:wq`. La frase atribuida a `:x` tambien guarda y sale, pero solo si hay cambios.

## Comodines de la shell

Los comodines (_wildcards_) son patrones que la shell expande antes de ejecutar un comando. La expansion se produce en el orden de evaluacion de la linea de comandos, despues de la expansion de variables pero antes de la ejecucion. El comando nunca ve el comodin; recibe la lista de archivos que coinciden con el patron.

### El comodin `*`

El asterisco representa cualquier secuencia de caracteres, incluida la cadena vacia:

```bash
ls *.txt       # Todos los archivos terminados en .txt
ls informe*    # Todos los archivos que empiecen por "informe"
ls *           # Todos los archivos del directorio actual (ocultos no)
```

La expansion del asterisco incluye archivos en el directorio actual unicamente. No atraviesa subdirectorios a menos que se combine con patrones de ruta:

```bash
ls **/*.txt    # Con globstar activado, recursivo sobre subdirectorios
```

### El comodin `?`

La interrogacion representa exactamente un caracter cualquiera:

```bash
ls documento?.txt    # documento1.txt, documentoA.txt, pero no documento10.txt
ls ???.log           # Archivos .log con nombres de exactamente tres caracteres
```

### Los corchetes `[]`

Los corchetes definen un conjunto de caracteres, y coinciden con exactamente uno de ellos:

```bash
ls [abc]*.txt        # Archivos .txt que empiecen por a, b o c
ls informe[0-9].md   # informe0.md, informe1.md, ..., informe9.md
ls [a-z]*.conf       # Archivos .conf que empiecen por letra minuscula
```

Dentro de los corchetes se pueden usar rangos (`[0-9]`, `[a-z]`, `[A-Z]`) y negacion con `!` al inicio (`[!0-9]*` coincide con archivos que no empiecen por digito).

### Combinacion de comodines

Los comodines se combinan para crear patrones precisos:

```bash
# Archivos que empiezan por vocal y terminan en .md
ls [aeiou]*.md

# Archivos con exactamente 3 caracteres antes de la extension
ls ???.*

# Archivos de log de enero a marzo (01, 02, 03)
ls error_0[1-3].log
```

### Peligros de la expansion de comodines

La expansion ocurre antes de la ejecucion del comando, lo que puede tener consecuencias inesperadas:

- Si no hay archivos que coincidan con el patron, la shell pasa el comodin literal (dependiendo de la configuracion de `nullglob`). Por defecto en bash, `ls *.inexistente` ejecuta `ls` con el argumento literal `*.inexistente`, y `ls` informa de que el archivo no existe.
- Los nombres de archivo con espacios se dividen en argumentos separados a menos que se cite la expansion. Esto es relevante al usar comodines en bucles:

```bash
# Incorrecto: si "mi archivo.txt" existe, se divide en dos argumentos
for f in *.txt; do
    rm $f   # Elimina "mi" y "archivo.txt" por separado
done

# Correcto: se cita la variable
for f in *.txt; do
    rm "$f"
done
```

> La opcion de shell `nullglob` (`shopt -s nullglob`) evita que los patrones sin coincidencias se pasen como literales. Con `nullglob` activado, si no hay archivos `.txt`, el bucle `for f in *.txt` simplemente no se ejecuta. Sin el, el bucle ejecuta una iteracion con `f="*.txt"`, que probablemente no es lo que se desea.

