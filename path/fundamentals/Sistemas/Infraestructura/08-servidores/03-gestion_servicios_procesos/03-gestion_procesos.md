# Gestion de procesos

Cada programa que se ejecuta en Linux se materializa como un proceso, una entidad con su propio espacio de direcciones, permisos y ciclo de vida. La capacidad de inspeccionar, controlar y priorizar estos procesos es fundamental para mantener un servidor estable y responder eficientemente ante situaciones como consumo excesivo de CPU, procesos huérfanos o servicios que no responden.

## El ciclo de vida de un proceso

Un proceso en Linux nace cuando un proceso padre ejecuta la llamada al sistema `fork()`, creando una copia de si mismo. El proceso hijo puede entonces ejecutar un nuevo programa con `exec()`, reemplazando su codigo en memoria. Todos los procesos descienden jerarquicamente del proceso init (PID 1, que en sistemas modernos es systemd).

Un proceso puede estar en diferentes estados a lo largo de su existencia:

- **Running** (R): esta ejecutandose activamente en la CPU o listo para hacerlo.
- **Sleeping** (S): esta esperando un evento (entrada de red, lectura de disco, signal). Es el estado mas comun en servidores.
- **Stopped** (T): ha recibido la signal `SIGSTOP` o `SIGTSTP` y se encuentra pausado.
- **Zombie** (Z): ha terminado su ejecucion pero su proceso padre aun no ha recogido su estado con `wait()`. No consume recursos pero ocupa una entrada en la tabla de procesos.
- **Disk Sleep** (D): esta bloqueado esperando operaciones de E/S de disco, un estado que no puede ser interrumpido.

Cuando un proceso termina, se envia la signal `SIGCHLD` al proceso padre. Si el padre no la recoge (no ejecuta `wait()`), el proceso hijo se convierte en zombie hasta que el padre termine o sea adoptado por init.

## Inspeccionar procesos activos

### ps: instantanea de procesos

`ps` proporciona un snapshot del estado de los procesos en el momento de su ejecucion. A diferencia de `top`, no se actualiza periodicamente.

```bash
ps aux                    # Todos los procesos en formato extendido
ps -ef                    # Todos los procesos en formato Unix clasico
ps aux | grep nginx       # Filtrar procesos de nginx
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%cpu | head -20  # Top 20 por CPU
```

Los campos mas relevantes son `PID` (identificador del proceso), `PPID` (proceso padre), `%CPU` y `%MEM` (consumo de recursos). La columna `STAT` indica el estado actual del proceso.

### top: monitor en tiempo real

`top` muestra una vista actualizada cada pocos segundos de los procesos y el estado general del sistema. La parte superior muestra metricas globales: carga del sistema, procesos totales, uso de CPU desglosado (usuario, sistema, idle, wait) y consumo de memoria y swap.

Las teclas de navegacion mas utiles:

- `M`: ordenar por uso de memoria.
- `P`: ordenar por uso de CPU.
- `1`: mostrar cada nucleo de CPU por separado.
- `k`: enviar una signal a un proceso (matar o reniciar).
- `q`: salir.

### htop: alternativa mejorada

htop es una alternativa interactiva a top con soporte para colores, navegacion con raton y una representacion visual mas intuitiva del uso de CPU y memoria. Ademas permite buscar procesos por nombre y gestionarlos directamente sin necesidad de recordar PIDs.

```bash
htop
```

## Terminar procesos

### kill: enviar signals a procesos

`kill` no "mata" un proceso directamente, sino que le envia una signal. El comportamiento depende de la signal:

- **SIGTERM (15)**: solicita al proceso que termine ordenadamente. Es la signal por defecto y debe ser la primera opcion, ya que permite al proceso liberar recursos y cerrar conexiones.
- **SIGKILL (9)**: fuerza la terminacion inmediata. No puede ser ignorada ni atrapada por el proceso. Debe usarse solo cuando SIGTERM no funciona.
- **SIGHUP (1)**: tradicionalmente usada para recargar configuracion. Muchos demonios la interpretan como "releer config".
- **SIGSTOP (19)**: pausa el proceso. Se puede reanudar con `SIGCONT`.

```bash
kill 1234          # Envia SIGTERM al proceso 1234
kill -9 1234       # Envia SIGKILL (forzado)
kill -HUP 1234     # Envia SIGHUP (recargar config)
```

### pkill: terminar por nombre o patron

`pkill` permite enviar signals a procesos basandose en su nombre, usuario o otros atributos, sin necesidad de buscar el PID manualmente.

```bash
pkill nginx            # Matar todos los procesos llamados nginx
pkill -u www-data      # Matar todos los procesos del usuario www-data
pkill -f "python app.py"  # Matar procesos cuyo comando completo coincida
```

> Usar SIGKILL sin antes intentar SIGTERM es una mala practica. El proceso no podra ejecutar su rutina de limpieza, lo que puede causar archivos de bloqueo persistentes o datos incompletos.

## Prioridad de procesos: nice y renice

Linux asigna a cada proceso un valor de **niceness** que va de -20 (prioridad maxima) a 19 (prioridad minima). Un valor negativo indica que el proceso tiene prioridad sobre los demas, y un valor positivo indica que cede CPU a otros procesos.

`nice` lanza un nuevo programa con una prioridad modificada:

```bash
nice -n 10 ./tarea_pesada.sh    # Ejecutar con prioridad baja
nice -n -5 ./tarea_critica.sh   # Ejecutar con prioridad alta (requiere root)
```

`renice` modifica la prioridad de un proceso ya en ejecucion:

```bash
renice -n 15 -p 1234    # Reducir prioridad del proceso 1234
renice -n -10 -p 5678   # Aumentar prioridad (requiere root)
```

Es importante entender que nice no garantiza un porcentaje fijo de CPU. Es una sugercion al planificador: procesos con nice negativo obtienen preferencia cuando hay contention de CPU, pero si un nucleo esta libre, cualquier proceso puede usarlo independientemente de su niceness.

En servidores, es comun ejecutar tareas de mantenimiento (respaldos, indices de bases de datos, compilaciones) con nice elevado para que no afecten al rendimiento de los servicios activos.

