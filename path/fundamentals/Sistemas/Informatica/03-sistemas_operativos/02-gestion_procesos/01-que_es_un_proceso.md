# Qué es un proceso

Un proceso es una instancia de un programa en ejecución. Cuando un usuario ejecuta una aplicación, el sistema operativo crea un proceso que contiene el código del programa, los datos que este utiliza y el estado de ejecución en un momento dado. Los procesos son la unidad fundamental de trabajo en un sistema operativo multitarea: cada aplicación que se ejecuta, cada servicio del sistema y cada comando que se lanza desde la línea de comandos se materializa como uno o varios procesos.

La distinción entre programa y proceso es fundamental. Un programa es una entidad estática: un archivo almacenado en disco que contiene instrucciones. Un proceso es una entidad dinámica: una entidad en ejecución que tiene memoria asignada, un espacio de direcciones propio y un estado que cambia con el tiempo. El mismo programa puede dar lugar a múltiples procesos simultáneos, cada uno con su propio estado y datos.

## Diferencia entre programa y proceso

Un programa es como una receta de cocina: contiene las instrucciones pero no está haciendo nada. Un proceso es como el acto de cocinar: está ejecutando las instrucciones de la receta con ingredientes concretos en un momento dado. Dos personas pueden cocinar la misma receta (dos procesos del mismo programa) usando ingredientes diferentes y en momentos diferentes.

Técnicamente, un proceso se compone de:

- **Texto del código (*text segment*):** las instrucciones del programa, que son de solo lectura y compartidas entre procesos del mismo programa.
- **Datos (*data segment*):** variables globales y constantes del programa.
- **Montículo (*heap*):** memoria dinámica asignada durante la ejecución (con `malloc`, `new`, etc.).
- **Pila (*stack*):** variables locales, parámetros de funciones y direcciones de retorno.
- **Registros de la CPU:** el estado de los registros del procesador en el momento de la última ejecución del proceso.
- **Información de E/S:** archivos abiertos, dispositivos asignados y conexiones de red.

## Estado de un proceso

Un proceso atraviesa diferentes estados durante su ciclo de vida:

**Nuevo (*new*):** el proceso está siendo creado. El sistema operativo está asignando recursos y preparando la estructura de datos del proceso.

**Listo (*ready*):** el proceso está preparado para ejecutarse y espera asignación de CPU. El planificador de procesos selecciona de la cola de procesos listos al que ejecutará a continuación.

**Ejecutando (*running*):** el proceso está siendo ejecutado activamente por la CPU. Solo un proceso por núcleo de procesador puede estar en este estado en un momento dado.

**Esperando (*waiting/blocked*):** el proceso está esperando un evento externo: una operación de E/S, la llegada de datos por la red, la finalización de otro proceso o la expiración de un temporizador. No puede ejecutarse hasta que el evento ocurra.

**Terminado (*terminated*):** el proceso ha finalizado su ejecución. Sus recursos están siendo liberados o ya han sido liberados.

```
        ┌──────────┐
        │   Nuevo  │
        └────┬─────┘
             ↓
        ┌──────────┐   asignación    ┌──────────┐
        │  Listo   │ ←── de CPU ──── │Ejecutando│
        └────┬─────┘                 └────┬─────┘
             ↑                            ↓
             │     evento             ┌──────────┐
             └──── Esperando ←────── │ Esperando│
                  completado          └──────────┘
                                           ↓
                                     ┌──────────┐
                                     │Terminado │
                                     └──────────┘
```

## Tabla de procesos (PCB)

El PCB (*Process Control Block*) es la estructura de datos que el sistema operativo utiliza para almacenar toda la información sobre un proceso. El PCB contiene:

- **Identificador del proceso (PID):** número único que identifica al proceso.
- **Estado del proceso:** el estado actual (listo, ejecutando, esperando, etc.).
- **Contador de programa:** la dirección de la siguiente instrucción a ejecutar.
- **Registros de la CPU:** el estado de los registros al momento de la última descontextualización.
- **Información de memoria:** las tablas de páginas que definen el espacio de direcciones del proceso.
- **Información de E/S:** archivos abiertos, dispositivos asignados.
- **Información de planificación:** prioridad, tiempo de ejecución acumulado, tiempo de espera.

El PCB es la estructura que permite la multitarea: cuando el planificador decide alternar entre procesos, almacena el estado del proceso actual en su PCB y carga el estado del siguiente proceso desde su PCB, un proceso conocido como *context switch* (conmutación de contexto).

## Procesos padres e hijos

Los procesos en Unix/Linux se organizan en una jerarquía de árbol. Cada proceso tiene un proceso padre que lo creó, y puede crear múltiples procesos hijos. El proceso PID 1 (`init` o `systemd`) es el ancestro de todos los procesos del sistema.

Cuando un proceso ejecuta la llamada `fork()`, el sistema operativo crea una copia exacta del proceso (el hijo), que comparte el mismo código pero tiene su propio espacio de datos y estado. El proceso padre y el hijo pueden entonces ejecutarse concurrentemente, comunicarse a través de tuberías o memoria compartida, o coordinarse mediante señales.

La jerarquía de procesos tiene implicaciones para la gestión de recursos y la terminación. Cuando un proceso termina, sus procesos hijos pueden convertirse en *huérfanos* (adoptados por `init`/`systemd`) o ser terminados también dependiendo de la implementación del SO. El mecanismo *zombie* ocurre cuando un proceso hijo termina pero su proceso padre aún no ha leído su estado de terminación.

> El modelo de procesos padre-hijo de Unix es elegante pero puede ser confuso. `fork()` crea un proceso hijo que es una copia casi idéntica del padre, y luego `exec()` reemplaza el código del hijo con un nuevo programa. Esta combinación de `fork()+exec()` es la forma estándar de ejecutar nuevos programas en Unix.

