# Protección de memoria

La protección de memoria es el conjunto de mecanismos hardware y software que garantizan que cada proceso solo pueda acceder a su propio espacio de direcciones de memoria. Sin protección, un proceso con un error de programación (como un puntero NULL o un desbordamiento de buffer) podría corromper la memoria de otros procesos o del propio sistema operativo, causando inestabilidad o fallos de seguridad.

La protección se implementa a través de la combinación de la MMU, las tablas de páginas y los mecanismos de privilegio del procesador. Estos mecanismos son fundamentales para la estabilidad y seguridad de los sistemas multitarea.

## Protección a nivel de hardware

El hardware de protección se basa en la distinción entre modo usuario y modo kernel (modo supervisor). En modo usuario, el proceso solo puede acceder a su propio espacio de direcciones y ejecutar un subconjunto limitado de instrucciones. En modo kernel, el sistema operativo tiene acceso a todo el espacio de direcciones y puede ejecutar cualquier instrucción.

Los mecanismos de protección hardware incluyen:

- **Bit de modo:** indica si la CPU está en modo usuario o kernel. Las transiciones entre modos solo pueden ocurrir a través de mecanismos controlados (llamadas al sistema, interrupciones, excepciones).
- **Registros de límite de segmento:** en segmentación, cada registro de base/límite define los límites del espacio de direcciones accesible. Un acceso fuera de los límites genera una excepción.
- **Bits de protección en tablas de páginas:** cada entrada de la tabla de páginas tiene bits que indican si la página es de solo lectura, de ejecución, accesible solo en modo kernel, etc.
- **No-execute (*NX bit*) o XD (*eXecute Disable*):** marca las páginas de datos como no ejecutables, previniendo la ejecución de código inyectado por ataques de buffer overflow.

## Protección a nivel de software

El sistema operativo implementa protección a nivel de software utilizando los mecanismos hardware disponibles:

- **Aislamiento de procesos:** cada proceso tiene su propia tabla de páginas, y la MMU traduce las direcciones usando solo la tabla del proceso activo. Un proceso no puede acceder a la memoria de otro proceso a menos que se haya establecido explícitamente una región de memoria compartida.
- **Memoria protegida del kernel:** las páginas del espacio del kernel están mapeadas en cada tabla de páginas pero marcadas como accesibles solo en modo kernel. Si un proceso en modo usuario intenta acceder a memoria del kernel, se genera una excepción de protección (*segfault*).
- **ASLR (*Address Space Layout Randomization*):** aleatoriza la ubicación de las regiones de memoria (pila, montículo, bibliotecas compartidas) cada vez que se inicia un proceso, dificultando ataques de inyección de código que dependen de direcciones conocidas.
- **Stack canaries:** valores conocidos colocados antes del retorno de función en la pila. Si un buffer overflow sobrescribe el retorno, el canary se corrompe y el sistema detecta la manipulación antes de ejecutar el código de retorno.

## Protección contra ataques comunes

**Buffer overflow:** ocurre cuando un programa escribe más datos de los que caben en un buffer, sobrescribiendo memoria adyacente. Los mecanismos de protección incluyen NX bit (pila no ejecutable), stack canaries, ASLR y verificación de límites por el compilador.

**Return-oriented programming (ROP):** técnica que reutiliza fragmentos de código existente (*gadgets*) para construir ataques, evitando la inyección de código. ASLR dificulta este ataque al aleatorizar las direcciones de las bibliotecas.

**Use-after-free:** acceder a memoria que ya ha sido liberada. Las protecciones incluyen verificación de punteros por el compilador (Address Sanitizer) y la aleatorización de la asignación de memoria.

**Double-free:** liberar dos veces la misma región de memoria. Puede corrompir las estructuras de datos del asignador de memoria y permitir ejecución arbitraria de código.

> La protección de memoria es una responsabilidad compartida entre el hardware (proceso, MMU), el sistema operativo (gestión de tablas de páginas, modo kernel) y las herramientas de compilación (stack canaries, sanitizadores). Ningún nivel por sí solo es suficiente; la protección se logra mediante la combinación de todas estas capas.

