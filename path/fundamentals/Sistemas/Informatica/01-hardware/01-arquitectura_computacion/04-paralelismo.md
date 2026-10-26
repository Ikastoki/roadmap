# Paralelismo y rendimiento

Un solo núcleo de procesador ejecuta una instrucción a la vez, en un flujo secuencial que se repite miles de millones de veces por segundo. Esta velocidad es impresionante, pero tiene un límite físico: la velocidad de la luz impone restricciones sobre la frecuencia máxima de reloj, y la disipación de calor limita cuántos transistores pueden trabajar simultáneamente en un espacio determinado. El paralelismo emerge como la solución a estas restricciones: en lugar de ejecutar una sola tarea más rápido, se ejecutan múltiples tareas al mismo tiempo.

El rendimiento de un sistema no se mide exclusivamente por la velocidad de su procesador. Depende de múltiples factores: la velocidad de la memoria, la eficiencia del sistema operativo, la arquitectura del software y la naturaleza paralela o secuencial de las tareas que se ejecutan. Comprender cómo el paralelismo interactúa con estos factores es fundamental para diseñar sistemas eficientes.

## Paralelismo a nivel de instrucción

El paralelismo a nivel de instrucción (ILP, *Instruction-Level Parallelism*) se refiere a la capacidad de un procesador de ejecutar múltiples instrucciones simultáneamente dentro de un mismo programa. Existen varias técnicas que aprovechan este paralelismo:

**Pipelining:** como se vio en la sección anterior, el pipelining superpone las fases de ejecución de múltiples instrucciones. Un pipeline de cinco etapas permite tener cinco instrucciones en proceso simultáneamente, aunque cada una individualmente sigue tomando cinco ciclos. El rendimiento efectivo se aproxima a una instrucción completada por ciclo.

**Ejecución fuera de orden:** los procesadores modernos pueden reordenar las instrucciones para maximizar el uso de los recursos del procesador. Si una instrucción depende de un dato que aún no está disponible (un *data hazard*), el procesador puede ejecutar instrucciones posteriores que no tengan esa dependencia, siempre que el resultado final sea idéntico al de una ejecución secuencial. Esta técnica, implementada por primera vez en el Intel Pentium Pro (1995), aumentó significativamente el rendimiento sin alterar la semántica del programa.

**Ejecución especulativa:** el procesador predice qué instrucciones se ejecutarán probablemente y comienza a ejecutarlas antes de confirmar la predicción. Si la predicción es correcta, se gana tiempo. Si es incorrecta, se deshacen los resultados y se vuelve al camino correcto. La predicción de saltos (*branch prediction*) es uno de los mecanismos más críticos: los procesadores modernos alcanzan precisiones superiores al 95% en la predicción de saltos, lo que reduce significativamente los penalties de control.

**Superscalaridad:** un procesador superscalar puede ejecutar múltiples instrucciones en un mismo ciclo de reloj, utilizando múltiples unidades de ejecución (ALUs, FPU, unidades de carga/almacenamiento). El número de instrucciones que se pueden ejecutar simultáneamente se denomina *issue width* y varía entre 2 y 8 en los procesadores comerciales actuales.

> El paralelismo a nivel de instrucción está limitado por las dependencias entre instrucciones. Si la instrucción B necesita el resultado de la instrucción A, no se puede ejecutar antes que ella. Estas dependencias reducen el paralelismo real por debajo del teórico, especialmente en código con muchos cálculos secuenciales.

## Multitarea y multiprocesamiento

Mientras que el paralelismo a nivel de instrucción opera dentro de un solo hilo de ejecución, la multitarea y el multiprocesamiento abordan el paralelismo a nivel de programa: múltiples tareas o procesos se ejecutan simultáneamente, ya sea en un solo núcleo (multitarea) o en múltiples núcleos (multiprocesamiento).

**Multitarea:** el sistema operativo alterna rápidamente entre múltiples procesos, concediendo a cada uno un fragmento de tiempo de CPU. El usuario percibe que todos se ejecutan simultáneamente, aunque en realidad solo uno está activo en cada momento. Esta ilusión se logra mediante los *context switches*, cambios de contexto que guardan el estado de un proceso y restauran el estado del siguiente.

**Multiprocesamiento simétrico (SMP):** múltiples núcleos de procesador comparten la misma memoria y el sistema operativo distribuye las tareas entre ellos de forma equilibrada. Cada núcleo puede ejecutar un proceso independiente, lo que permite un paralelismo real. La mayoría de los ordenadores modernos —desde smartphones hasta servidores— utilizan arquitecturas SMP.

**Multiprocesamiento asimétrico (AMP):** los núcleos no son iguales. Algunos son más rápidos y potentes (núcleos de alto rendimiento) mientras que otros son más eficientes energéticamente (núcleos de baja potencia). Esta arquitectura, común en dispositivos móviles, permite al sistema asignar las tareas críticas a los núcleos potentes y las tareas de fondo a los núcleos eficientes.

El desafío fundamental del multiprocesamiento es la gestión de la concurrencia. Cuando múltiples núcleos acceden a la misma zona de memoria simultáneamente, se producen condiciones de carrera (*race conditions*) que pueden corromper los datos. Los mecanismos de sincronización —mutexes, semáforos, bloqueos— resuelven estos problemas pero introducen overhead y pueden reducir el paralelismo si se utilizan de forma inadecuada.

> La Ley de Amdahl establece que el speedup máximo obtenible al paralelizar un programa está limitado por la fracción secuencial del mismo. Si el 10% de un programa es inherentemente secuencial, el máximo speedup posible es 10x, sin importar cuántos núcleos se añadan. Esta ley tiene implicaciones profundas para el diseño de software paralelo.

## Conceptos básicos de rendimiento

El rendimiento de un sistema informático se evalúa mediante múltiples métricas que capturan diferentes aspectos de su comportamiento:

**Tiempo de ejecución (latencia):** el tiempo total que un sistema tarda en completar una tarea. Es la métrica más directa pero también la más ambigua: incluye el tiempo de ejecución en CPU, el tiempo de espera en memoria, el tiempo de acceso a disco y el tiempo de E/S. Para comparar sistemas, se utiliza comúnmente el *wall clock time*, el tiempo real transcurrido desde el inicio hasta el fin de la tarea.

**Throughput (rendimiento):** el número de tareas completadas por unidad de tiempo. Un servidor que procesa 1000 peticiones por segundo tiene mayor throughput que uno que procesa 500, aunque el tiempo de respuesta individual sea similar. Throughput y latencia no siempre están correlacionados: un sistema puede tener bajo throughput pero baja latencia, o viceversa.

**CPI (Cycles Per Instruction):** el número promedio de ciclos de reloj necesarios para ejecutar una instrucción. Un CPI de 1 significa que cada instrucción se completa en un ciclo de reloj. Los procesadores superscalar pueden alcanzar CPI menores a 1 ejecutando múltiples instrucciones por ciclo. El CPI es una métrica útil para comparar eficiencias entre procesadores con diferentes frecuencias de reloj.

**IPC (Instructions Per Cycle):** el inverso del CPI. Representa el número promedio de instrucciones completadas por ciclo de reloj. Un IPC de 4 significa que el procesador completa en promedio 4 instrucciones por ciclo, lo que indica una arquitectura superscalar eficiente.

**Frecuencia de reloj:** mide el número de ciclos por segundo que ejecuta el procesador, expresado en hercios (Hz). Un procesador a 3.5 GHz ejecuta 3.500 millones de ciclos por segundo. La frecuencia es importante pero no determina por sí sola el rendimiento: un procesador a 3 GHz con IPC de 3 es más rápido que uno a 4 GHz con IPC de 1.

**Ancho de banda:** el volumen de datos que se puede transferir por unidad de tiempo. Se aplica a buses, memorias y dispositivos de E/S. Un bus de memoria con un ancho de banda de 50 GB/s puede transferir 50 gigabytes de datos por segundo, lo que es crítico para cargas de trabajo que requieren acceso intensivo a la memoria.

> La relación entre frecuencia, IPC y número de núcleos determina el rendimiento total de un procesador. Un núcleo rápido a 5 GHz con IPC alto puede ser más eficiente para tareas secuenciales que cuatro núcleos a 3 GHz con IPC bajo, dependiendo de la naturaleza de la carga de trabajo.

