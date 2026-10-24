# Sistema de ayuda integrado

Linux incorpora documentacion completa de todos los comandos, las llamadas al sistema, los formatos de archivo y mucho mas. Aprender a navegar esta documentacion es tan importante como conocer los comandos mismos: ningun administrador recuerda todas las opciones de todas las herramientas, pero todo administrador sabe como encontrar la informacion que necesita en el momento preciso.

## help: ayuda para comandos internos

Los comandos internos de la shell incorporan su propia documentacion accesible mediante `help`. Esta ayuda es especifica de la shell y solo funciona con builtins. No depende de paginas de manual externas ni de archivos en el sistema.

```bash
# Ayuda del comando interno cd
help cd

# Ayuda con formato mas completo
help help

# Listar todos los comandos internos disponibles
help
```

La salida de `help` describe la sintaxis, las opciones disponibles y, en algunos casos, proporciona ejemplos. Es una ayuda concisa, disenada para consulta rapida desde la terminal. Para comandos externos, `help` no funciona; en su lugar se usa `--help` como opcion del propio comando, una convencion establecida por GNU.

```bash
# Opcion de ayuda para comandos externos
ls --help
grep --help
tar --help
```

La mayoria de comandos GNU implementan `--help` y muestran un resumen de uso junto con las opciones principales. Es el primer recurso cuando se necesita recordar una opcion concreta sin salir de la terminal.

## man: paginas del manual

El sistema `man` (manual) es la documentacion tradicional de Unix. Organiza la informacion en paginas estructuradas accesibles mediante el comando `man` seguido del nombre del tema.

```bash
# Abrir la pagina del manual de ls
man ls

# Buscar en paginas del manual por palabra clave
man -k "system calls"
```

La navegacion dentro de una pagina del manual utiliza las teclas de movimiento del paginador `less` (que es el visor por defecto):

- `Enter` o `j`/`k`: avanzar/retroceder una linea.
- `Espacio` o `Page Up`/`Page Down`: avanzar/retroceder una pantalla.
- `/` (barra): buscar texto hacia adelante.
- `n` y `N`: siguiente/anterior coincidencia de busqueda.
- `q`: salir de la pagina del manual.

Cada pagina del manual sigue una estructura estandar con las siguientes secciones:

- **NAME**: nombre del comando y breve descripcion.
- **SYNOPSIS**: sintaxis de uso, con opciones entre corchetes y argumentos obligatorios sin ellos.
- **DESCRIPTION**: descripcion detallada del funcionamiento.
- **OPTIONS**: listado completo de opciones con su explicacion.
- **EXIT STATUS**: codigos de salida y su significado.
- **ENVIRONMENT**: variables de entorno que afectan al comportamiento.
- **FILES**: archivos que utiliza o referencia el comando.
- **EXAMPLES**: ejemplos de uso (no siempre presente).
- **SEE ALSO**: referencias a otras paginas del manual relacionadas.

## Secciones del manual

El manual de Unix esta dividido en secciones numeradas que organizan el contenido por tipo. El mismo nombre puede aparecer en varias secciones con significados distintos. Por ejemplo, `printf` existe como comando (seccion 1) y como llamada al sistema en C (seccion 3).

| Seccion | Contenido | Ejemplo |
|---------|-----------|---------|
| 1 | Comandos de usuario | `ls(1)`, `grep(1)` |
| 2 | Llamadas al sistema | `fork(2)`, `open(2)` |
| 3 | Funciones de biblioteca en C | `printf(3)`, `malloc(3)` |
| 4 | Dispositivos especiales y drivers | `null(4)`, `tty(4)` |
| 5 | Formatos de archivo y protocolos | `passwd(5)`, `crontab(5)` |
| 6 | Juegos y demos | `tetris(6)` |
| 7 | Convenciones, caracteres especiales y miscelanea | `ascii(7)`, `regex(7)` |
| 8 | Comandos de administracion | `mount(8)`, `useradd(8)` |

Para acceder a una seccion concreta, se indica el numero antes del nombre:

```bash
# Ver la pagina de la llamada al sistema open (seccion 2)
man 2 open

# Ver el formato del archivo passwd (seccion 5)
man 5 passwd

# Ver el comando passwd (seccion 1)
man 1 passwd
```

Si no se especifica seccion, `man` muestra la primera pagina que encuentra, que suele ser la de menor numero (tipicamente la seccion 1). El orden de busqueda puede configurarse con la variable de entorno `MANSECT`.

> La seccion 5 es especialmente util para los administradores, ya que documenta el formato de archivos de configuracion como `/etc/passwd`, `/etc/crontab` o `/etc/ssh/sshd_config`. Cuando se necesita saber exactamente que campos componen un archivo de configuracion, `man 5 nombre_archivo` es la referencia oficial.

```bash
# Ejemplos practicos de consulta
man -k "copy files"     # Busca paginas relacionadas con copia de archivos
man -f ls               # Muestra descripcion breve de ls (seccion y resumen)
```

## info: documentacion GNU

El proyecto GNU desarrollo su propio sistema de documentacion, `info`, que ofrece hypertexto y estructura jerarquica, a diferencia del formato plano de `man`. Aunque menos utilizado hoy en dia, ciertos programas GNU proporcionan documentacion mas extensa en formato info que en man.

```bash
# Abrir la documentacion info de bash
info bash

# Navegacion basica en info
# n: siguiente nodo, p: nodo anterior, u: nivel superior
# Tab: saltar al siguiente enlace, Enter: seguir enlace
# q: salir
```

La mayoria de los usuarios prefiere `man` por su simplicidad y rapidez, pero `info` resulta util cuando se necesita documentacion muy detallada, como la del sistema de construccion Autotools o el editor Emacs.

> Una regla practica: probar primero `comando --help` para una consulta rapida, luego `man comando` para la documentacion completa, y finalmente `info comando` si se necesita el nivel mas alto de detalle. Las paginas del manual de `man` suelen ser suficientes para el trabajo diario.

