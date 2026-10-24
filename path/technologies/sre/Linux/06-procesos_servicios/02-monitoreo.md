# Monitoreo de procesos

Observar el estado del sistema en tiempo real es la primera linea de diagnostico ante cualquier anomalia. Linux proporciona herramientas que van desde la instantanea estatica de `ps` hasta el dashboard interactivo de `htop`. Elegir la herramienta adecuada depende de la situacion: un analisis forense requiere `ps` con formato personalizado, un pico de CPU necesita `top` con ordenacion dinamica, y una sesion de depuracion prolongada se beneficia de `htop` y su navegacion con teclado.

## ps: instantanea de procesos

El comando `ps` captura el estado de los procesos en el momento exacto de su ejecucion. No actualiza la informacion: es una fotografia, no una pelicula. Su potencia reside en la capacidad de formatear la salida para extraer exactamente los campos que interesan.

```bash
# Lista completa de procesos del sistema (estilo BSD)
ps aux

# Lista en formato jerarquico (estilo Unix)
ps -ef

# Columnas personalizadas
ps -eo pid,ppid,user,%cpu,%mem,stat,comm --sort=-%cpu

# Arbol de dependencias
ps -ejH
```

La combinacion `ps aux` muestra: usuario (USER), PID, uso de CPU (%), uso de memoria (%), tamano de memoria virtual (VSZ), tamano de memoria residente (RSS), estado (STAT), terminal (TTY), tiempo de CPU acumulado (TIME) y comando (COMMAND). El estado aparece como una cadena de caracteres donde la primera letra indica el estado base (R, S, D, T, Z) y las siguientes indican caracteristicas adicionales como prioridad (< para alta prioridad, N para baja) o si el proceso tiene paginada su memoria principal (W).

```bash
# Filtrar procesos de un usuario especifico
ps -u usuario

# Procesos asociados a una terminal
ps -t pts/0

# Formato con campos de memoria
ps -eo pid,comm,rss,vsz --sort=-rss | head -10
```

> `ps aux` es el primer comando que un administrador ejecuta al conectarse a un servidor desconocido. Revela que esta corriendo, quien lo ejecuta y cuanto consume.

## top: monitor dinamico

`top` actualiza la informacion en tiempo real por defecto cada 3 segundos. Presenta un resumen del sistema en las primeras lineas: uptime, numero de usuarios, carga media del sistema (load average en 1, 5 y 15 minutos), total de procesos, uso de CPU (usuario, sistema, nice, idle, espera de E/S, interrupciones de hardware y software, y steal en virtualizacion), uso de memoria fisica y swap.

```bash
# Iniciar top con intervalo de 1 segundo
top -d 1

# Ordenar por uso de memoria (dentro de top: pulsar M)
# Ordenar por uso de CPU (dentro de top: pulsar P)
# Mostrar solo procesos de un usuario (dentro de top: pulsar u)

# Mostrar procesos de un PID concreto
top -p 1234,5678
```

Las columnas de `top` en su vista predeterminada incluyen: PID, usuario (USER), prioridad (PR), valor nice (NI), tamano virtual (VIRT), tamano residente (RES), tamano de memoria compartida (SHR), estado (S), uso de CPU (%), uso de memoria (%), tiempo de CPU acumulado (TIME+) y comando (COMMAND).

Dentro de `top`, las teclas mas utiles son:
- `P`: ordenar por uso de CPU (descendente)
- `M`: ordenar por uso de memoria
- `k`: enviar senal a un proceso (pide PID y senal)
- `r`: cambiar la prioridad (renice)
- `u`: filtrar por usuario
- `c`: mostrar la linea de comandos completa
- `q`: salir

> La carga media del sistema (load average) no es el uso de CPU. Es la longitud media de la cola de procesos en estado ejecutable o en E/S no interrumpible. Un load de 4.0 en un sistema de 4 nucleos significa colas equilibradas; en un sistema de 2 nucleos indica saturacion.

## htop: monitor interactivo mejorado

`htop` no viene instalado por defecto en todas las distribuciones (requiere `sudo apt install htop`), pero ofrece una experiencia significativamente superior a `top`. Muestra barras de colores para CPU y memoria, permite desplazamiento vertical y horizontal con las teclas de cursor, incluye un arbol de procesos navegable y permite enviar senales con una interfaz de menu.

```bash
# Iniciar htop
htop

# Arbol de procesos
htop -t

# Filtrar por nombre de proceso (F3 dentro de htop)
# Ordenar por columna (F6 dentro de htop)
# Matar proceso (F9 dentro de htop)
```

Caracteristicas diferenciadoras de `htop`:
- Las barras de CPU muestran la carga de cada nucleo por separado
- La seccion de memoria distingue entre memoria usada, buffers y cache
- El arbol de procesos (F5) visualiza la relacion padre-hijo
- La busqueda incremental (F3) resalta procesos mientras se escribe
- No necesita `sudo` para que el usuario vea sus propios procesos

```bash
# htop con filtro desde la linea de comandos
htop --filter=nginx
```

> `htop` es la herramienta preferida para sesiones de diagnostico prolongadas. Su interfaz colorida reduce la carga cognitiva al presentar la informacion de forma visual en lugar de tablas densas.

## Otras herramientas de monitoreo

El ecosistema de monitoreo de Linux incluye herramientas especializadas para escenarios concretos:

```bash
# atrop: similar a top pero enfocado en E/S de disco
sudo iotop

# Ver uso de E/S por proceso
sudo iotop -o

# Monitoreo de red por proceso
sudo nethogs

# Estadisticas de CPU, memoria y E/S por proceso
pidstat 2

# Uso de archivos abiertos
lsof
```

