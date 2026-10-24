# Procesos y estados

Un proceso es una instancia en ejecucion de un programa. Cuando el kernel carga un binario en memoria, le asigna un espacio de direcciones virtuales, una serie de recursos (archivos abiertos, senales pendientes, variables de entorno) y un bloque de control (task_struct en el codigo fuente de Linux). Cada proceso es una entidad aislada: los fallos en uno no deberian, en teoria, comprometer la estabilidad del sistema. Esta separacion es la base del modelo multitarea de Unix, donde cientos de procesos coexisten compitiendo por el mismo CPU y la misma memoria fisica.

## PID y PPID

Cada proceso recibe un identificador unico numerico llamado PID (Process ID). Este numero se asigna de forma secuencial hasta alcanzar el limite definido en `/proc/sys/kernel/pid_max` (tradicionalmente 32768, hoy configurable hasta 4194304). Linux reutiliza los PID una vez que el proceso termina y su estado es recolectado por el padre, pero nunca asigna el mismo PID a dos procesos vivos simultaneamente.

El PPID (Parent Process ID) es el PID del proceso que lo creo mediante la llamada al sistema `fork()`. Todo proceso en Linux tiene un padre excepto el proceso init (PID 1) lanzado directamente por el kernel durante el arranque. Esta relacion parental forma un arbol jerarquico que puede visualizarse con `pstree`.

```bash
# Ver PID y PPID de los procesos actuales
ps -eo pid,ppid,comm

# Arbol de procesos
pstree -p

# PID del proceso actual desde la shell
echo $$
# PID del proceso padre de la shell
echo $PPID
```

> Los PID bajos (1-300 aproximadamente) pertenecen a procesos del sistema creados durante el arranque: systemd, kthreadd, y sus descendientes directos. Matar estos procesos desestabiliza el sistema.

## Estados de un proceso

El estado de un proceso refleja en que punto del ciclo de ejecucion se encuentra respecto al planificador del kernel. Linux define varios estados, pero los cuatro mas relevantes para la administracion del sistema son:

**Running (R).** El proceso esta ejecutandose en la CPU o esta en la cola de listos para ejecutar. En sistemas con un solo nucleo, solo un proceso puede estar realmente en ejecucion; el resto espera su turno en la cola. El planificador (CFS, Completely Fair Scheduler) decide cual sera el siguiente basandose en una proporcion de tiempo justo ponderada por la prioridad del proceso (nice value).

**Sleeping (S).** El proceso esta bloqueado esperando que ocurra un evento: una operacion de E/S, un temporizador, una senal, o la liberacion de un recurso. Dormir es eficiente porque el proceso no consume ciclos de CPU. Se subdivide en sueño interrumpible (S, puede despertarse con una senal) y sueño no interrumpible (D, espera E/S directa y no responde a senales hasta que la operacion concluya). Los procesos en estado D suelen indicar problemas con el disco o el sistema de archivos.

**Stopped (T).** La ejecucion del proceso se ha detenido por accion de una senal (SIGSTOP o SIGTSTP). El proceso permanece en memoria, conserva sus recursos y puede reanudarse con SIGCONT. Es el estado tipico de un trabajo suspendido con Ctrl+Z.

**Zombie (Z).** El proceso ha terminado pero su entrada en la tabla de procesos aun existe porque el padre no ha ejecutado `wait()` para recoger su codigo de salida. Los zombies no consumen CPU ni memoria significativa, pero ocupan entradas en la tabla de procesos. Un padre que nunca llama a `wait()` genera zombies persistentes; si el padre muere, los hijos zombies son adoptados por init (PID 1), que periodicamente ejecuta `wait()` para limpiarlos.

```bash
# Ver procesos con su estado
ps aux

# Contar procesos por estado
ps -eo stat | sort | uniq -c

# Identificar procesos zombies
ps aux | awk '{if ($8 == "Z") print}'
```

## Ciclo de vida de un proceso

El nacimiento de un proceso sigue siempre el mismo patron: un proceso existente invoca `fork()`, que duplica el espacio de memoria del padre creando un proceso hijo. El hijo recibe una copia de las variables, descriptores de archivo y contexto de senales del padre. Inmediatamente despues, el hijo suele llamar a `exec()` para reemplazar su propia imagen de memoria con un nuevo programa.

```bash
# Simulacion conceptual del ciclo fork + exec
# fork() crea una copia; exec() carga un nuevo programa
strace -f -e trace=process bash -c "ls"
```

El proceso vive en memoria mientras ejecuta codigo. Cuando termina (por llamada a `exit()`, recepcion de una senal fatal o fallo grave), su codigo de salida queda almacenado en la tabla de procesos. El kernel envia SIGCHLD al padre para notificarle que el hijo ha terminado. Solo cuando el padre ejecuta `wait()` (o `waitpid()`), el kernel libera la entrada completa de la tabla de procesos y el zombie desaparece.

> Un proceso que muere sin que su padre recoja el codigo de salida se convierte en zombie. En condiciones normales, los zombies desaparecen rapidamente. Si persists, revisa el codigo del proceso padre: probablemente no gestiona correctamente SIGCHLD.

## El proceso init y la jerarquia

El proceso init (tradicionalmente SysV, hoy systemd con PID 1) es el ancestro comun de todos los procesos del sistema. Su responsabilidad principal es adoptar procesos huerfanos (cuyo padre ha muerto antes que ellos) y mantener el sistema en un estado coherente. Si el PID 1 muere, el kernel entra en panico.

```bash
# Ver el arbol completo de procesos
pstree -p

# El proceso kthreadd (PID 2) es el ancestro de todos
# los hilos del kernel
cat /proc/2/comm
```

