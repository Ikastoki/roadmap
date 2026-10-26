# Jerarquía y flujo de datos

La arquitectura de Von Neumann establece los componentes fundamentales de una computadora, pero no explica cómo interactúan entre sí para completar un trabajo. La jerarquía del sistema describe la organización de los componentes según su función y velocidad, mientras que el flujo de datos describe cómo la información se mueve entre ellos durante la ejecución de un programa. Comprender ambos conceptos es esencial para entender por qué un sistema rinde de cierta forma y dónde se producen los cuellos de botella.

La jerarquía no es solo una clasificación estática: refleja una realidad física. Los componentes más rápidos son también los más caros y los de menor capacidad, lo que obliga a un diseño equilibrado donde los datos se mueven entre niveles según su frecuencia de uso. El flujo de datos, por su parte, es dinámico: cambia con cada instrucción que se ejecuta y depende de factores como la arquitectura del procesador, la velocidad de la memoria y la eficiencia del sistema operativo.

## Jerarquía del sistema

Los componentes de una computadora se organizan en una jerarquía que refleja sus características de velocidad, capacidad y coste. Desde la CPU hasta el almacenamiento permanente, cada nivel intermedio actúa como un amortiguador que reduce la disparidad de velocidades entre componentes adyacentes.

### Procesamiento

En la cúspide de la jerarquía se encuentra la unidad de procesamiento central (CPU). La CPU ejecuta las instrucciones del programa y coordina las operaciones de todos los demás componentes. Su velocidad se mide en gigahercios (GHz), lo que equivale a miles de millones de ciclos de reloj por segundo. Cada ciclo permite a la CPU ejecutar una o más operaciones básicas, como sumar dos números o leer un byte de memoria.

Dentro de la CPU existen registros de alta velocidad que almacenan los datos más utilizados. Estos registros son extremadamente rápidos —accesibles en un solo ciclo de reloj— pero su capacidad es muy limitada, generalmente de unos pocos cientos de bytes. Los registros son el nivel más rápido y pequeño de toda la jerarquía de memoria.

### Memoria

La memoria principal (RAM) se sitúa un nivel por debajo de la CPU en la jerarquía. Ofrece un equilibrio entre velocidad y capacidad: es mucho más rápida que un disco duro, pero más lenta que los registros de la CPU. La RAM típica tiene tiempos de acceso del orden de los 10-100 nanosegundos, frente a los sub-nanosegundo de los registros.

La memoria se organiza en celdas direccionables, cada una capaz de almacenar una palabra de datos (típicamente 32 o 64 bits). El procesador accede a la memoria mediante direcciones, que son números que identifican la posición de cada celda. La velocidad de la memoria es un factor crítico del rendimiento del sistema, ya que el procesador debe esperar a que los datos estén disponibles antes de ejecutar cada instrucción.

> La relación entre la velocidad de la CPU y la de la memoria ha sido uno de los desafíos más persistentes de la informática. Mientras que los procesadores han aumentado su velocidad exponencialmente, la memoria ha mejorado a un ritmo mucho más lento, creando un desfase que obliga a utilizar múltiples niveles de cache.

### Almacenamiento

El almacenamiento permanente —discos duros (HDD) y unidades de estado sólido (SSD)— ocupa la base de la jerarquía. Es el nivel más lento pero también el de mayor capacidad y el único que conserva los datos cuando el sistema se apaga.

Los discos duros mecánicos tienen tiempos de acceso del orden de los milisegundos, mientras que los SSD basados en memoria flash los reducen a microsegundos. Aún así, ambos son significativamente más lentos que la RAM, lo que convierte al almacenamiento en un cuello de botella potencial cuando el sistema necesita leer o escribir grandes volúmenes de datos.

La capacidad de almacenamiento se mide en terabytes (TB) en los sistemas modernos, lo que permite guardar grandes colecciones de datos, programas y sistemas operativos. Sin embargo, la velocidad de acceso es tan inferior a la de la RAM que el sistema operativo debe gestionar cuidadosamente qué datos se mantienen en memoria y cuáles se mantienen en disco.

### Entrada y salida (I/O)

Los dispositivos de entrada y salida forman la interfaz entre la computadora y el mundo exterior. Su posición en la jerarquía varía enormemente según el dispositivo: un SSD NVMe conectado por PCIe puede alcanzar velocidades cercanas a la RAM, mientras que un teclado USB tiene requisitos de latencia mucho más modestos.

La gestión de los dispositivos de E/S es una de las tareas más complejas del sistema operativo. Cada dispositivo tiene sus propias características de velocidad, protocolo y formato de datos, lo que requiere un software intermedio —los drivers— que traduzca las operaciones del sistema operativo a instrucciones comprensibles por el dispositivo.

## La interconexión de los principales componentes

CPU, memoria y dispositivos de E/S no operan de forma aislada. Están conectados a través de una red de buses y controladores que permiten el intercambio de datos, direcciones y señales de control.

El bus del sistema es el canal principal de comunicación. En su forma más simple, consta de tres componentes:

- **Bus de datos:** transporta la información propiamente dicha entre la CPU, la memoria y los dispositivos de E/S. Su anchura (típicamente 32 o 64 bits) determina cuántos datos se pueden transferir en un solo ciclo.
- **Bus de direcciones:** transporta las direcciones de memoria que el procesador desea leer o escribir. Su anchura determina el espacio de direccionamiento máximo del sistema: un bus de 32 bits puede direccionar hasta 4 GB de memoria, mientras que un bus de 64 bits permite direccionar cantidades astronómicas.
- **Bus de control:** transporta las señales que coordinan las operaciones, como las peticiones de lectura/escritura, las interrupciones y las señales de sincronía.

En los sistemas modernos, estos buses se han diversificado en múltiples canales especializados. El bus PCIe conecta los dispositivos de alta velocidad como las GPU y los SSD NVMe, mientras que buses más lentos como USB y SATA atienden dispositivos con menores requisitos de rendimiento.

## Ciclo de instrucción

El ciclo de instrucción es el proceso mediante el cual la CPU ejecuta cada instrucción de un programa. Es el corazón del funcionamiento de un procesador y su comprensión es fundamental para entender cómo se traducen las instrucciones de un programa en operaciones físicas.

### Fetch, Decode, Execute

El ciclo básico consta de tres fases principales:

**Fetch (Búsqueda):** la CPU lee la siguiente instrucción de la memoria. La dirección de la instrucción se almacena en un registro especial llamado *Program Counter* (PC) o *Instruction Pointer* (IP). Después de leer la instrucción, el PC se incrementa para apuntar a la siguiente.

**Decode (Decodificación):** la instrucción leída se descompone en sus componentes: el código de operación (qué acción realizar) y los operandos (sobre qué datos actuar). La unidad de control interpreta estos campos y genera las señales necesarias para ejecutar la operación.

**Execute (Ejecución):** la ALU u otro componente de la CPU realiza la operación especificada. Esto puede involucrar aritmética, lógica, acceso a memoria o transferencia de datos entre registros.

En muchos procesadores modernos existe una cuarta fase:

**Writeback (Escritura):** el resultado de la operación se almacena en el registro de destino o en la memoria. Esta fase cierra el ciclo y prepara al procesador para la siguiente instrucción.

### Papel de la CPU, memoria y buses

Durante cada fase del ciclo, diferentes componentes del sistema trabajan de forma coordinada:

- En la fase **fetch**, la CPU envía la dirección del Program Counter a través del bus de direcciones y recibe la instrucción a través del bus de datos. La memoria debe responder en un tiempo determinado para no retrasar el ciclo.
- En la fase **decode**, la CPU opera internamente, sin necesidad de acceder a la memoria ni a los dispositivos de E/S. Esta fase es completamente local al procesador.
- En la fase **execute**, la ALU puede necesitar datos adicionales de la memoria o de los registros. Si los datos no están en los registros internos, la CPU debe acceder a la memoria, lo que introduce una latencia adicional.
- En la fase **writeback**, el resultado se escribe en la memoria o en un registro, completando el ciclo.

> La eficiencia del ciclo de instrucción depende en gran medida de la velocidad de acceso a la memoria. Un procesador rápido puede ejecutar la fase decode y execute en un solo ciclo, pero si la memoria no puede proporcionar los datos a esa velocidad, el procesador se ve obligado a esperar, desperdiciando ciclos de reloj.

### Pipelining y sus beneficios

El *pipelining* es una técnica que permite superponer múltiples instrucciones en diferentes fases de ejecución. En lugar de esperar a que una instrucción complete todas sus fases antes de comenzar la siguiente, el procesador puede iniciar la fase fetch de la segunda instrucción mientras la primera está en la fase decode, y así sucesivamente.

En un pipeline de cinco etapas (fetch, decode, execute, memory, writeback), el procesador puede tener hasta cinco instrucciones en diferentes fases simultáneamente. Esto no reduce el tiempo de ejecución de cada instrucción individual, pero aumenta significativamente el número de instrucciones completadas por unidad de tiempo.

El pipeline introduce complejidades adicionales. Los *hazards* (peligros) ocurren cuando una instrucción depende del resultado de otra que aún no ha completado su ejecución. Los procesadores modernos utilizan técnicas como la reordenación de instrucciones, la ejecución fuera de orden y la predicción de saltos para minimizar el impacto de estos hazards.

### Ejemplo práctico simplificado

Consideremos la ejecución de la instrucción `ADD R1, R2, R3` que suma los valores de los registros R2 y R3 y almacena el resultado en R1:

```
Ciclo 1 (Fetch):    PC -> Dirección de memoria -> Lee instrucción ADD
Ciclo 2 (Decode):   Decodifica: operación=ADD, operandos=R2, R3, destino=R1
Ciclo 3 (Execute):  ALU suma R2 + R3
Ciclo 4 (Writeback): Resultado se escribe en R1
```

Sin pipelining, cada instrucción debe esperar a que la anterior termine completamente. Con pipelining, mientras la instrucción ADD está en la fase execute, la siguiente instrucción puede estar en decode y la siguiente en fetch, triplicando potencialmente el número de instrucciones completadas por segundo.

### Principios de localidad (temporal y espacial)

Los principios de localidad explican por qué los sistemas jerárquicos funcionan tan bien en la práctica. Existen dos tipos fundamentales de localidad:

**Localidad temporal:** los datos que han sido accedidos recientemente tienden a ser accedidos de nuevo en un futuro cercano. Este principio justifica la existencia de caches: si un dato se acaba de usar, es probable que se vuelva a usar pronto, por lo que tiene sentido mantenerlo en un nivel rápido de la jerarquía.

**Localidad espacial:** los datos almacenados en direcciones de memoria cercanas a un dato accedido tienden a ser accedidos en un futuro cercano. Este principio justifica que las caches y los sistemas operativos transfieran datos en bloques (líneas de cache, páginas de memoria) en lugar de byte por byte.

> Los principios de localidad son la razón por la que los caches funcionan tan bien en la práctica. Sin localidad, un cache de 32 KB sería inútil frente a una memoria de 16 GB. Pero gracias a la localidad, un cache pequeño puede capturar la gran mayoría de los accesos a memoria, reduciendo drásticamente los tiempos promedio de acceso.

