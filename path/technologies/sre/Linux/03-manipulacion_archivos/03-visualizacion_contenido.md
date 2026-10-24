# Visualizacion de contenido de archivos

Los archivos de texto plano son la moneda de cambio de la administracion de sistemas Linux: configuracion, logs, scripts, documentacion y datos estructurados residen en este formato. La linea de comandos ofrece un espectro de herramientas para inspeccionar ese contenido, desde la volcacion completa con `cat` hasta la paginacion interactiva con `less`, pasando por la visualizacion parcial con `head` y `tail`. Cada una esta optimizada para un caso de uso concreto, y la eleccion incorrecta puede significar desde una terminal bloqueada por un archivo binario hasta la imposibilidad de localizar una entrada en un log de varios gigabytes.

## Volcado completo con cat

`cat` (_concatenate_) lee uno o varios archivos secuencialmente y escribe su contenido en la salida estandar. Es la herramienta mas directa para visualizar archivos pequenos, pero su utilidad va mas alla de la mera inspeccion.

```bash
cat archivo.txt
```

Cuando se pasan multiples archivos, `cat` los concatena en el orden en que se especifican:

```bash
cat parte1.txt parte2.txt parte3.txt > documento_completo.txt
```

`cat` es, ante todo, una herramienta de redireccion. Su uso para visualizar archivos en la terminal queda relegado a ficheros de pocas decenas de lineas; para archivos extensos, el texto se desplaza fuera de la pantalla sin posibilidad de retroceder.

Las opciones mas relevantes incluyen:

- `-n`: numera todas las lineas.
- `-b`: numera solo las lineas no vacias.
- `-s`: suprime lineas repetidas vacias, colapsando multiples lineas en blanco en una sola.
- `-E`: muestra un simbolo `$` al final de cada linea (visible para detectar espacios finales).
- `-T`: muestra los tabuladores como `^I`.

```bash
cat -n script.sh
#     1  #!/bin/bash
#     2  echo "Hola"
#     3
#     4  exit 0
```

> En entornos donde un archivo puede contener caracteres no imprimibles (como binarios o secuencias de escape de terminal), es recomendable usar `cat -v` o directamente `cat -A` (equivalente a `-vET`), que muestra los caracteres de control en notacion visible (`^X` para caracteres de control, `M-` para bytes con el bit alto activado).

## Paginacion interactiva con less

`less` es un paginador: muestra el contenido de un archivo por pantallas y permite navegar hacia adelante y hacia atras. Nacio como una mejora de `more`, del que toma el nombre (el juego de palabras _less is more_ es intencionado).

```bash
less archivo_grande.log
```

Una vez dentro de `less`, la navegacion se realiza mediante teclas:

| Tecla | Accion |
|-------|--------|
| `Espacio` / `f` | Avanzar una pagina |
| `b` | Retroceder una pagina |
| `j` / `Flecha abajo` | Avanzar una linea |
| `k` / `Flecha arriba` | Retroceder una linea |
| `g` | Ir al principio del archivo |
| `G` | Ir al final del archivo |
| `/patron` | Buscar hacia adelante |
| `?patron` | Buscar hacia atras |
| `n` / `N` | Siguiente/anterior resultado de busqueda |
| `q` | Salir |

La busqueda hacia adelante con `/` resalta todas las coincidencias en el archivo y permite navegar entre ellas con `n`. La busqueda no distingue entre mayusculas y minusculas si el patron comienza con mayusculas; para busqueda insensible explicitamente se usa `-i` como opcion de arranque o `I` dentro de la sesion.

```bash
less -i archivo.log
# Busqueda insensible a mayusculas/minusculas
```

`less` no carga el archivo completo en memoria. Lee el contenido bajo demanda a medida que el usuario navega, lo que permite inspeccionar archivos de gigabytes sin consumir memoria proporcional al tamano del archivo. Internamente, `less` utiliza llamadas al sistema `lseek()` para desplazarse por el archivo sin leer datos innecesarios.

Las opciones de arranque mas utiles son:

- `-N`: muestra numeros de linea.
- `-S`: no divide las lineas largas; permite desplazamiento horizontal con las flechas izquierda/derecha.
- `-F`: sale automaticamente si el archivo cabe en una pantalla (comportamiento similar a `cat`).
- `+F`: modo _follow_, similar a `tail -f`; actualiza la pantalla cuando el archivo crece.

```bash
less -N -S archivo_con_lineas_largas.csv
```

> La opcion `+F` convierte `less` en una herramienta de monitoreo de logs en tiempo real. Al alcanzar el final del archivo, `less` se queda en espera y muestra las nuevas lineas a medida que se anaden. Para salir del modo _follow_ y volver a la navegacion normal, se pulsa `Ctrl+C`.

## El predecesor more

`more` fue el paginador original de Unix. Hoy en dia esta practicamente desplazado por `less`, pero sigue disponible en la mayoria de los sistemas:

```bash
more archivo.txt
```

La diferencia fundamental es que `more` solo permite avanzar. No hay tecla para retroceder una pagina ni busqueda hacia atras. La navegacion se limita a `Espacio` (avanzar pagina), `Enter` (avanzar linea) y `q` (salir).

En las distribuciones modernas, `more` suele ser un enlace simbolico a `less` ejecutado con opciones que imitan el comportamiento limitado de `more`. Esto significa que, en la practica, `more` puede ofrecer funcionalidades de `less` sin que el usuario lo sepa.

La unica ventaja real de `more` sobre `less` es su presencia en sistemas minimos o entornos de recuperacion donde `less` no esta instalado. En cualquier otro caso, `less` es la opcion superior.

## Visualizar el principio con head

`head` muestra las primeras lineas de un archivo. Por defecto, son 10 lineas:

```bash
head archivo.csv
```

La opcion `-n` permite especificar un numero distinto de lineas:

```bash
head -n 20 archivo.csv
# Muestra las primeras 20 lineas

head -n 5 *.csv
# Muestra las primeras 5 lineas de cada archivo .csv con su cabecera
```

Una sintaxis abreviada acepta el numero directamente tras el guion:

```bash
head -20 archivo.csv
# Equivalente a head -n 20
```

`head` es util para inspeccionar la estructura de un archivo antes de procesarlo: verificar el formato de las columnas en un CSV, comprobar el encoding de caracteres o confirmar que el archivo no esta corrupto. En tuberias, `head` sirve para limitar la salida de comandos que generan mucho texto:

```bash
dmesg | head -30
# Primeras 30 lineas del log del kernel
```

La opcion `-q` (_quiet_) suprime las cabeceras que `head` anade cuando recibe multiples archivos, y `-v` (_verbose_) las fuerza incluso para un solo archivo.

## Visualizar el final con tail

`tail` es el complemento simetrico de `head`: muestra las ultimas lineas de un archivo, 10 por defecto:

```bash
tail archivo.log
```

La opcion `-n` funciona igual que en `head`:

```bash
tail -n 50 archivo.log
# Ultimas 50 lineas
```

La utilidad mas potente de `tail` es la opcion `-f` (_follow_). En este modo, `tail` no termina tras mostrar las ultimas lineas; permanece en ejecucion y muestra las nuevas lineas a medida que se anaden al archivo:

```bash
tail -f /var/log/syslog
# Muestra las nuevas entradas del log en tiempo real
```

`tail -f` es indispensable para el monitoreo de logs en vivo. Cuando se combina con `grep`, permite filtrar eventos especificos mientras ocurren:

```bash
tail -f /var/log/nginx/access.log | grep " 500 "
# Muestra solo las peticiones que devuelven error 500
```

Internamente, `tail -f` utiliza `inotify` en Linux moderno para recibir notificaciones del kernel cuando el archivo cambia. En sistemas sin `inotify`, `tail` recurre a un sondeo periodico (polling) que verifica el tamano del archivo cada segundo, un metodo menos eficiente.

La opcion `-F` (mayuscula) extiende `-f` con la capacidad de seguir el archivo incluso si este se rota (truncado o reemplazado). Es la opcion recomendada para logs gestionados por `logrotate`, donde el archivo se renombra y se crea uno nuevo:

```bash
tail -F /var/log/auth.log
# Sigue el archivo incluso tras rotacion
```

> La diferencia entre `-f` y `-F` es critica en entornos de produccion. `-f` deja de mostrar datos si el archivo se rota, porque el descriptor de archivo abierto apunta al archivo antiguo (renombrado). `-F` detecta que el inodo ha cambiado y reabre automaticamente el nuevo archivo.

## Combinacion de head y tail

`head` y `tail` pueden combinarse mediante tuberias para extraer una seccion especifica de un archivo. Para mostrar las lineas 50 a 60:

```bash
head -n 60 archivo.txt | tail -n 11
```

El razonamiento: `head -n 60` obtiene las primeras 60 lineas; `tail -n 11` extrae las ultimas 11 de esas 60, que corresponden a las lineas 50-60. Esta tecnica es util para inspeccionar secciones concretas de archivos grandes sin cargarlos en un editor.

