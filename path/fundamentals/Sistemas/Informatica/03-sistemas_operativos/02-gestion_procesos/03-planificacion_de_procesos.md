# Planificación de procesos

La planificación de procesos es la función del sistema operativo que determina qué proceso se ejecutará a continuación y durante cuánto tiempo. En un sistema multitarea, la CPU puede ejecutar solo un proceso por núcleo en un momento dado, pero hay múltiples procesos listos para ejecutarse. El planificador resuelve este conflicto de recursos distribuyendo el tiempo de la CPU entre los procesos de forma eficiente, justa y responsiva.

El diseño del planificador tiene un impacto directo en la experiencia del usuario (tiempo de respuesta), el rendimiento del sistema (throughput) y la eficiencia en el uso de la CPU (utilización). Un buen planificador balancea estos objetivos a menudo conflictivos.

## Planificador de corto, medio y largo plazo

**Planificador de largo plazo (*job scheduler*):** decide qué procesos se admiten en el sistema para su ejecución. Controla el grado de multiprogramación (número de procesos en memoria). En sistemas de tiempo compartido, el planificador de largo plazo ejecuta con poca frecuencia, ya que los procesos se admiten rápidamente.

**Planificador de corto plazo (*CPU scheduler*):** decide qué proceso de la cola de listos se ejecutará a continuación. Opera con frecuencia muy alta (cada pocos milisegundos o menos) y es el responsable directo del rendimiento percibido del sistema.

**Planificador de medio plazo (*swapper*):** decide qué procesos se cargan en memoria y cuáles se intercambian (*swap out*) al disco. Opera a frecuencia intermedia y equilibra la carga entre memoria y disco.

## Algoritmos de planificación

**FCFS (*First-Come, First-Served*):** los procesos se ejecutan en el orden en que llegaron a la cola de listos. Es el algoritmo más simple pero puede causar el *efecto convoy*: un proceso largo bloquea a todos los que llegaron después, incluso si son cortos y interactivos.

**SJF (*Shortest Job First*):** selecciona el proceso con la menor duración estimada de ráfaga de CPU. Optimiza el tiempo promedio de espera pero requiere conocer (o estimar) la duración futura del proceso, lo que es impráctico en muchos casos.

**Round Robin (*RR*):** cada proceso recibe un *quantum* de tiempo (típicamente 10-100 ms). Cuando el quantum expira, el proceso se desaloja y se coloca al final de la cola de listos, dando paso al siguiente. Proporciona tiempos de respuesta bajos y justicia entre procesos, pero la elección del quantum es crítica: demasiado corto implica overhead excesivo por conmutaciones de contexto; demasiado largo se comporta como FCFS.

**Prioridades:** cada proceso tiene una prioridad asignada y se ejecuta el de mayor prioridad. Los problemas incluyen la inanición (*starvation*) de procesos de baja prioridad, que se resuelve con *envejecimiento* (*aging*): gradualmente incrementar la prioridad de procesos que llevan mucho tiempo esperando.

**Colas multinivel (*Multilevel Feedback Queue*):** combina múltiples colas con diferentes prioridades y quantums. Los procesos nuevos entran en la cola de mayor prioridad; si consumen todo su quantum, descienden a una cola de menor prioridad. Permite que los procesos interactivos (cortos) se ejecuten rápidamente mientras los procesos de larga duración se ejecutan con menor prioridad.

## Quantum de tiempo y conmutación de contexto

El **quantum de tiempo** es el período máximo que un proceso puede ejecutarse continuamente antes de ser desalojado por el planificador. Un quantum típico en Linux es de 6 ms, lo que permite ejecutar miles de procesos por segundo con tiempos de respuesta perceptiblemente bajos.

La **conmutación de contexto** (*context switch*) es el proceso de guardar el estado del proceso actual (registros, contador de programa, tabla de páginas) en su PCB y cargar el estado del siguiente proceso. El coste de una conmutación de contexto varía según el hardware pero típicamente es de 1-10 microsegundos, lo que puede sumar un overhead significativo si ocurre con demasiada frecuencia.

> El diseño del planificador es un equilibrio constante entre throughput y latencia. Un planificador que maximiza el throughput permite que el sistema procese el mayor número de trabajos posible, pero puede sacrificar los tiemmos de respuesta para aplicaciones interactivas. Un planificador que minimiza la latencia garantiza respuestas rápidas pero puede reducir el rendimiento global.

## Multiprogramación y multiprocesamiento

**Multiprogramación:**允许多os procesos residan en memoria simultáneamente, permitiendo que la CPU alternne entre ellos. Cuando un proceso se bloquea esperando E/S, la CPU ejecuta otro proceso, maximizando la utilización del procesador.

**Multiprocesamiento simétrico (*SMP*):** la CPU tiene múltiples núcleos, y el sistema operativo distribuye los procesos entre los núcleos disponibles. Es el estándar en los sistemas modernos, donde un procesador de 8 núcleos puede ejecutar 8 procesos simultáneamente (o más con hyperthreading).

**Multiprocesamiento asimétrico (*AMP*):** los núcleos son diferentes (algunos de alto rendimiento, otros de bajo consumo) y el sistema operativo asigna procesos a los núcleos más adecuados según su carga de trabajo. Los diseños big.LITTLE de ARM y los híbridos de Intel (P-cores/E-cores) utilizan esta aproximación.

