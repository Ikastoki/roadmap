# Componentes internos de la CPU

Una CPU moderna contiene miles de millones de transistores organizados en subsistemas especializados que cooperan para ejecutar instrucciones. Estos componentes internos no son entidades separadas físicamente: están interconectados a través de buses de alta velocidad y comparten el espacio del chip para optimizar la latencia y el consumo energético. Comprender cada componente es esencial para entender por qué un procesador rinde de cierta forma y dónde se producen los cuellos de botella.

## Núcleos

El núcleo (*core*) es la unidad de procesamiento básica dentro de una CPU. Cada núcleo es esencialmente un procesador independiente capaz de ejecutar su propio hilo de instrucciones. Los procesadores modernos pueden contener desde 2 núcleos en CPUs básicas hasta 64 o más en procesadores de servidor.

La evolución hacia núcleos múltiples se produjo a mediados de la década de 2000, cuando los fabricantes encontraron límites físicos para aumentar la frecuencia de un solo núcleo. En lugar de hacer un núcleo más rápido, decidieron poner varios núcleos en el mismo chip, permitiendo la ejecución paralela de múltiples tareas.

Cada núcleo contiene sus propios registros, ALU, unidad de control y caches internos (L1 y L2). Los núcleos se comunican entre sí a través de un bus interno de alta velocidad y comparten el cache de nivel 3 (L3) y el controlador de memoria.

> El rendimiento de un sistema no depende solo del número de núcleos, sino de cómo el software aprovecha ese paralelismo. Un programa secuencial no se ejecutará más rápido en un procesador de 16 núcleos que en uno de 8, ya que solo utilizará un núcleo a la vez.

## Hilos

Un hilo (*thread*) es la unidad básica de ejecución que un sistema operativo puede programar en un núcleo. Cada hilo representa un flujo secuencial de instrucciones que se ejecutan de forma independiente.

**Hilos físicos:** corresponden directamente a núcleos de hardware. Un procesador de 8 núcleos puede ejecutar 8 hilos físicos simultáneamente.

**Hilos lógicos:** tecnologías como Intel Hyper-Threading o AMD SMT (*Simultaneous Multi-Threading*) permiten que un núcleo físico ejecute dos hilos lógicos alternando entre ellos. Cuando un hilo está esperando datos de la memoria, el núcleo puede ejecutar instrucciones del segundo hilo, mejorando la utilización del hardware.

El número óptimo de hilos depende de la carga de trabajo. Las aplicaciones de renderizado 3D, codificación de video y servidores de bases de datos se benefician significativamente de múltiples hilos. Las aplicaciones de gaming, por el contrario, suelen depender más de la velocidad single-core y del número de núcleos físicos.

## Unidad de Control (CU)

La unidad de control es el director de orquesta del núcleo. Lee las instrucciones de la memoria, las decodifica y genera las señales de control necesarias para que los demás componentes ejecuten la operación correspondiente.

En procesadores modernos, la CU es extraordinariamente sofisticada:

- **Decodificador de instrucciones:** traduce las instrucciones del conjunto ISA a microoperaciones internas comprensibles por el hardware.
- **Gestión de interrupciones:** prioriza y despacha las interrupciones que llegan desde los dispositivos de E/S.
- **Control del pipeline:** gestiona el flujo de instrucciones a través de las etapas del pipeline, resolviendo dependencias y prediciendo saltos.

## Unidad Aritmético-Lógica (ALU)

La ALU es el componente que realiza todas las operaciones matemáticas y lógicas. Cada núcleo puede contener múltiples ALU para ejecutar varias operaciones simultáneamente:

- **ALU entera:** opera sobre números enteros (suma, resta, multiplicación, división, operaciones bitwise).
- **FPU (*Floating-Point Unit*):** realiza operaciones sobre números de punto flotante, esenciales para cálculos científicos, gráficos e IA.
- **Unidades vectoriales:** procesan múltiples datos con una sola instrucción (SIMD). Las extensiones AVX-512 de Intel y AVX2 de AMD pueden procesar 16 o 32 valores de 32 bits simultáneamente.
- **Unidades criptográficas:** implementan instrucciones dedicadas para cifrado y descifrado AES, acelerando operaciones de seguridad.

## Unidades de control y ejecución

Más allá de la ALU y la CU, los procesadores modernos incorporan unidades especializadas:

**Unidad de predicción de saltos (*Branch Predictor*):** analiza el historial de ejecución para predecir qué caminos tomará el programa cuando encuentre instrucciones condicionales. Una predicción correcta permite mantener el pipeline lleno; una incorrecta obliga a descartar las instrucciones ejecutadas especulativamente, desperdiciando ciclos de reloj.

**Unidad de renombramiento de registros:** permite que el procesador reorganice las instrucciones para maximizar el uso de los recursos, renombrando registros físicos para evitar falsas dependencias.

**Unidades de carga y almacenamiento (*Load/Store Units*):** gestionan el movimiento de datos entre la memoria y los registros internos. Estas unidades son especialmente críticas ya que el acceso a la memoria es significativamente más lento que la ejecución en la ALU.

## Reloj

El reloj (*clock*) es una señal periódica que sincroniza todas las operaciones del procesador. Cada pulso del reloj marca el inicio de un ciclo de ejecución, y todos los componentes del procesador operan sincronizados con esta señal.

La frecuencia del reloj se mide en hercios (Hz) y representa el número de ciclos por segundo. Un procesador a 3.5 GHz completa 3.500 millones de ciclos por segundo. Dentro de cada ciclo, el procesador ejecuta una o más operaciones, dependiendo de la complejidad de la instrucción y la eficiencia del diseño.

> La frecuencia del reloj no es la única medida de rendimiento. Un procesador a 3 GHz con un IPC (*Instructions Per Cycle*) de 4 es dos veces más rápido que uno a 3 GHz con IPC de 2, ya que ejecuta el doble de instrucciones por ciclo.

## IPC (Instructions Per Cycle)

El IPC mide el número promedio de instrucciones completadas por ciclo de reloj. Es una métrica más reveladora que la frecuencia sola, ya que captura la eficiencia del diseño del procesador.

El IPC depende de múltiples factores:

- **Anchura del pipeline:** procesadores más anchos pueden decodificar y ejecutar más instrucciones simultáneamente.
- **Ejecución fuera de orden:** permite reordenar instrucciones para maximizar el uso de los recursos.
- **Tamaño de los caches:** caches más grandes reducen los tiempos de espera por acceso a la memoria.
- **Eficiencia de la predicción de saltos:** mejores predicciones significan menos pipeline flushes y menos ciclos desperdiciados.

Los procesadores modernos alcanzan IPC superiores a 5 en cargas de trabajo ideales, lo que significa que completan más de 5 instrucciones por ciclo de reloj en promedio. Sin embargo, el IPC real varía significativamente según la naturaleza del código: las cargas de trabajo con muchas dependencias de datos tienen menor IPC que las independientes.

