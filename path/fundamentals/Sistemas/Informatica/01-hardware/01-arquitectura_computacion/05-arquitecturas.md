# Tipos de arquitectura

El término arquitectura de procesador se refiere al diseño interno que determina cómo un chip ejecuta instrucciones, gestiona la memoria y se comunica con otros componentes. No existe una única forma de construir un procesador: las diferentes arquitecturas representan distintos compromisos entre rendimiento, consumo energético, complejidad de fabricación y compatibilidad con software existente. Elegir una arquitectura sobre otra tiene implicaciones que se extienden desde el nivel del silicio hasta las decisiones de diseño de software.

Las arquitecturas no son estáticas. Evolucionan continuamente para adaptarse a nuevas exigencias: la demanda de dispositivos móviles eficientes impulsó el desarrollo de arquitecturas de bajo consumo, mientras que la inteligencia artificial y el computo de alto rendimiento han generado nuevas extensiones y especializaciones.

## Qué son las arquitecturas

Una arquitectura de procesador define el contrato entre el hardware y el software. Especifica qué instrucciones puede ejecutar el procesador, cómo se representan los datos, cómo se accede a la memoria y cómo se comunican los componentes internos. Este conjunto de especificaciones se conoce como ISA (*Instruction Set Architecture*) y es la interfaz que el software debe cumplir para funcionar correctamente en un hardware determinado.

El ISA define elementos como:

- Los registros disponibles para el programador.
- Los tipos de datos soportados (enteros, punto flotante, vectores).
- Las instrucciones aritméticas, lógicas y de control disponibles.
- El modelo de memoria (cómo se direccionan las celdas).
- La gestión de interrupciones y excepciones.

Una misma ISA puede tener múltiples implementaciones físicas. El x86, por ejemplo, es implementado por Intel y AMD con diseños internos muy diferentes, pero ambos ejecutan el mismo conjunto de instrucciones, lo que garantiza compatibilidad binaria entre los dos.

> La distinción entre ISA y microarquitectura es fundamental. El ISA es el contrato visible para el programador; la microarquitectura es cómo el fabricante lo implementa internamente. Dos procesadores con el mismo ISA pueden tener rendimientos, consumo energético y costes radicalmente distintos.

## Arquitectura x86 vs ARM

Las dos arquitecturas dominantes en la computación actual son x86 y ARM, cada una con sus propias fortalezas y áreas de aplicación.

### x86

La arquitectura x86 fue introducida por Intel en 1978 con el procesador 8086. Desde entonces, ha sido el estándar de la computación de escritorio y servidores, con compatibilidad binaria que se ha mantenido durante más de cuatro décadas. Intel y AMD son los dos principales fabricantes de procesadores x86, y su ecosistema de software es extraordinariamente amplio.

x86 es una arquitectura CISC (*Complex Instruction Set Computer*) que ofrece un conjunto de instrucciones rico y flexible. Las instrucciones pueden ser de longitud variable (de 1 a 15 bytes) y pueden operar directamente sobre valores en memoria, lo que simplifica ciertos tipos de operaciones. Sin embargo, esta flexibilidad tiene un coste: la decodificación de instrucciones de longitud variable es más compleja y consume más energía.

Los procesadores x86 modernos son extremadamente potentes. Los Intel Core y AMD Ryzen de última generación pueden alcanzar frecuencias superiores a 5 GHz, ejecutar múltiples instrucciones por ciclo y gestionar caches de hasta 100 MB. Sin embargo, su consumo energético también es significativo, lo que los hace menos adecuados para dispositivos móviles con batería limitada.

### ARM

ARM (*Advanced RISC Machines*) es una arquitectura RISC (*Reduced Instruction Set Computer*) diseñada para ser eficiente energéticamente. A diferencia de x86, ARM utiliza instrucciones de longitud fija (32 bits en ARM clásico, 16 o 32 bits en Thumb), lo que simplifica la decodificación y reduce el consumo de energía.

ARM no fabrica procesadores directamente. En su lugar, licencia el diseño a otros fabricantes como Qualcomm, Apple, Samsung y MediaTek, que adaptan la arquitectura a sus necesidades. Esta estrategia de licenciamiento ha convertido a ARM en la arquitectura dominante en dispositivos móviles, con más del 99% de smartphones y tablets utilizando procesadores ARM.

En los últimos años, ARM ha expandido su presencia más allá de los dispositivos móviles. Los Apple Silicon (M1, M2, M3) han demostrado que ARM puede competir con x86 en rendimiento de escritorio, y los servidores basados en ARM (como los AWS Graviton) ofrecen una relación rendimiento/energía superior a los procesadores x86 en ciertas cargas de trabajo.

| Característica | x86 | ARM |
|---|---|---|
| Tipo de ISA | CISC | RISC |
| Longitud de instrucciones | Variable (1-15 bytes) | Fija (16/32 bits) |
| Consumo energético | Alto | Bajo |
| Rendimiento bruto | Muy alto | Alto y creciente |
| Mercado principal | Escritorio, servidores | Móvil, embebido, servidores |
| Fabricantes principales | Intel, AMD | Qualcomm, Apple, Samsung |

> La brecha entre x86 y ARM se ha reducido significativamente. Los procesadores ARM modernos incorporan técnicas como la decodificación compleja y la ejecución fuera de orden, mientras que los x86 han mejorado su eficiencia energética. La elección entre ambos hoy depende más del ecosistema de software y las necesidades específicas que de diferencias técnicas abismales.

## Sistemas de 32 y 64 bits

El término "bits" en el contexto de procesadores se refiere a dos conceptos relacionados pero distintos: el ancho de los registros de la CPU y el ancho del bus de direcciones de memoria.

**Procesadores de 32 bits:** los registros de la CPU y el bus de direcciones manejan palabras de 32 bits. Esto limita el espacio de direccionamiento a 2^32 direcciones, equivalentes a 4 GB de memoria. Para muchas aplicaciones, esta capacidad es insuficiente, especialmente en servidores y estaciones de trabajo que manejan grandes volúmenes de datos.

**Procesadores de 64 bits:** amplían los registros y el bus de direcciones a 64 bits, permitiendo direccionar teóricamente hasta 16 exabytes de memoria (2^64 bytes). En la práctica, los sistemas actuales utilizan solo 48 o 52 bits para el direccionamiento de memoria, lo que permite hasta 256 TB de espacio de direccionamiento, más que suficiente para cualquier aplicación actual.

La transición de 32 a 64 bits no solo aumentó la capacidad de direccionamiento. Los registros de 64 bits permiten trabajar con enteros de mayor precisión, lo que beneficia aplicaciones científicas, financieras y de bases de datos. Además, los procesadores de 64 bits suelen incluir más registros disponibles y extensiones de conjunto de instrucciones que mejoran el rendimiento en cargas de trabajo específicas.

> Los sistemas operativos de 64 bits pueden ejecutar programas de 32 bits en modo compatibilidad, pero lo contrario no es posible: un sistema operativo de 32 bits no puede aprovechar la capacidad completa de un procesador de 64 bits.

## CISC vs RISC

La distinción entre CISC y RISC representa dos filosofías de diseño opuestas que han influido profundamente en la evolución de los procesadores.

### CISC (Complex Instruction Set Computer)

Los procesadores CISC buscan acercar el lenguaje de máquina al lenguaje de alto nivel. Ofrecen instrucciones complejas que pueden ejecutar múltiples operaciones en un solo ciclo, como cargar un valor de memoria, sumarlo a un registro y escribir el resultado de vuelta en memoria, todo en una única instrucción.

Esta complejidad tiene ventajas: el código compilado puede ser más compacto (menos instrucciones para lograr lo mismo), y ciertas operaciones comunes se ejecutan de forma más eficiente. Sin embargo, las instrucciones complejas son más difíciles de decodificar y ejecutar, lo que puede limitar la frecuencia de reloj y la eficiencia energética.

Intel y AMD han mantenido la compatibilidad con el legado CISC de x86 mientras incorporan internamente técnicas RISC. Los procesadores x86 modernos traducen las instrucciones CISC a microoperaciones internas de estilo RISC, obteniendo lo mejor de ambos mundos.

### RISC (Reduced Instruction Set Computer)

Los procesadores RISC adoptan la filosofía opuesta: un conjunto de instrucciones pequeño y simple, donde cada instrucción se ejecuta en un solo ciclo de reloj. Las operaciones complejas se construyen combinando múltiples instrucciones simples.

La simplicidad de las instrucciones RISC facilita el diseño del hardware: la decodificación es más rápida, la ejecución es más predecible, y es más fácil implementar técnicas como el pipelining y la ejecución superscalar. Esto se traduce en mayor frecuencia de reloj y mejor eficiencia energética.

ARM, MIPS, RISC-V y PowerPC son arquitecturas RISC. RISC-V, en particular, ha ganado popularidad recientemente como una arquitectura RISC de código abierto, lo que permite a cualquier fabricante implementar procesadores sin pagar licencias.

### La convergencia

En la práctica, la distinción entre CISC y RISC se ha difuminado. Los procesadores CISC incorporan internamente técnicas RISC, y los procesadores RISC han añadido instrucciones complejas para mejorar el rendimiento en cargas de trabajo específicas. Lo que importa hoy no es la etiqueta CISC o RISC, sino cómo el procesador gestiona las dependencias, la memoria y la paralelidad a nivel de software.

## Arquitecturas paralelas

Más allá del paralelismo a nivel de instrucción, las arquitecturas paralelas buscan ejecutar múltiples operaciones simultáneamente a nivel de datos o de tareas. Flynn clasificó las arquitecturas de computación en cuatro categorías basándose en los flujos de instrucciones y datos:

**SISD (Single Instruction, Single Data):** un solo procesador ejecuta una instrucción sobre un dato a la vez. Es el modelo clásico de Von Neumann y sigue siendo la base de la computación secuencial.

**SIMD (Single Instruction, Multiple Data):** un solo procesador ejecuta la misma instrucción sobre múltiples datos simultáneamente. Las extensiones SSE y AVX de Intel, y las extensiones NEON de ARM, son ejemplos de SIMD. Son especialmente eficientes para procesamiento multimedia, gráficos y operaciones vectoriales.

**MISD (Multiple Instruction, Single Data):** múltiples procesadores ejecutan diferentes instrucciones sobre el mismo dato. Es una arquitectura teórica con poca aplicación práctica, aunque se ha utilizado en sistemas de alta disponibilidad donde se ejecutan múltiples algoritmos de verificación sobre los mismos datos.

**MIMD (Multiple Instruction, Multiple Data):** múltiples procesadores ejecutan diferentes instrucciones sobre diferentes datos simultáneamente. Es el modelo de los procesadores multinúcleo y los clusters de servidores, y representa la forma más común de paralelismo en la computación moderna.

## Arquitecturas híbridas

Los procesadores híbridos combinan núcleos de diferentes características en un solo chip. La arquitectura más común es la que mezcla núcleos de alto rendimiento (P-cores, por *Performance cores*) con núcleos de eficiencia energética (E-cores, por *Efficient cores*).

Intel introdujo esta arquitectura con la décima generación de Core (Alder Lake) en 2021, y Apple la había implementado previamente en sus chips M1 con núcleos Firestorm (alto rendimiento) e Icestorm (baja potencia). El sistema operativo asigna las tareas críticas a los núcleos potentes y las tareas de fondo a los núcleos eficientes, optimizando tanto el rendimiento como el consumo energético.

Esta aproximación es especialmente valiosa en dispositivos móviles, donde la batería es un recurso limitado. Un smartphone puede ejecutar una aplicación demandante en un núcleo P mientras mantiene la sincronización de correo electrónico en un núcleo E, consumiendo significativamente menos energía que si todos los núcleos fueran de alto rendimiento.

## Multinúcleo

La evolución hacia procesadores multinúcleo se aceleró a mediados de la década de 2000, cuando los fabricantes encontraron límites físicos para aumentar la frecuencia de reloj de un solo núcleo. En lugar de hacer un núcleo más rápido, decidieron poner varios núcleos en el mismo chip.

Los procesadores multinúcleo presentan desafíos únicos. El hardware debe garantizar la coherencia de memoria: cuando dos núcleos modifican la misma zona de memoria, el sistema debe resolver los conflictos para evitar inconsistencias. Los protocolos de coherencia como MESI (*Modified, Exclusive, Shared, Invalid*) y MOESI se encargan de esta tarea, pero introducen overhead y latencia.

El software también debe adaptarse al paralelismo. Los programas secuenciales no se benefician automáticamente de múltiples núcleos: deben reescribirse para distribuir el trabajo entre hilos. Los modelos de programación paralela —pthreads, OpenMP, MPI, CUDA— ofrecen herramientas para este propósito, pero la programación paralela sigue siendo significativamente más compleja que la secuencial.

> El número de núcleos en un procesador de consumo general ha crecido de 1 (a principios de los 2000) a 16 o más en los actuales. Los servidores pueden tener 64, 128 o incluso más núcleos. Sin embargo, el rendimiento por núcleo se ha estancado en los últimos años, lo que significa que el paralelismo real es la principal vía de mejora del rendimiento.

