# El ciclo de instrucción

El ciclo de instrucción es el proceso fundamental mediante el cual una CPU ejecuta el código de un programa. Cada instrucción del software —desde una suma simple hasta una operación criptográfica compleja— pasa por una serie de fases que transforman la instrucción codificada en binario en una operación física sobre los datos. Este ciclo se repite miles de millones de veces por segundo, y su eficiencia determina directamente el rendimiento del procesador.

Comprender el ciclo de instrucción es comprender cómo la máquina traduce el pensamiento del programador en acción. Cada fase del ciclo tiene un propósito específico, y los cuellos de botella en cualquiera de ellas retrasan la ejecución del programa completo.

## Detalle del ciclo de Fetch

El ciclo de búsqueda (*fetch*) es la primera fase del ciclo. Su objetivo es traer la siguiente instrucción de la memoria a los registros internos del procesador.

El proceso comienza con el registro *Program Counter* (PC), también llamado *Instruction Pointer* (IP), que contiene la dirección de memoria de la siguiente instrucción a ejecutar. La CPU envía esta dirección a través del bus de direcciones hacia la memoria principal.

Mientras la memoria prepara los datos, la CPU puede estar inactiva o ejecutando otra fase de una instrucción anterior (en sistemas con pipeline). La latencia de acceso a la memoria es el factor determinante de la velocidad de esta fase: si la instrucción está en el cache L1, se recupera en 1-4 ciclos; si debe buscarse en la memoria principal, pueden requerirse 50-100 ciclos o más.

Una vez recibida la instrucción, el Program Counter se incrementa para apuntar a la siguiente instrucción secuencial. Este incremento es automático y ocurre en paralelo con la decodificación de la instrucción actual.

> En arquitecturas con instrucciones de longitud variable como x86, el procesador no sabe dónde comienza la siguiente instrucción hasta que ha decodificado la actual. Esto introduce una complejidad adicional en el ciclo de fetch que no existe en arquitecturas con instrucciones de longitud fija como ARM.

## Decode

La fase de decodificación toma la instrucción cruda leída de la memoria y la descompone en sus componentes: el código de operación (qué acción realizar), los operandos (sobre qué datos actuar) y el registro destino (dónde almacenar el resultado).

En procesadores CISC como x86, la decodificación es especialmente compleja porque las instrucciones pueden tener longitudes variables y formatos diferentes. El decodificador debe interpretar correctamente la instrucción antes de poder generar las señales de control adecuadas. Para gestionar esta complejidad, los procesadores x86 modernos traducen las instrucciones CISC a microoperaciones internas de estilo RISC, que son más uniformes y fáciles de ejecutar.

En procesadores RISC como ARM, la decodificación es más sencilla: las instrucciones tienen un formato fijo y uniforme, lo que permite decodificarlas en un solo ciclo de reloj.

El resultado de la decodificación es un conjunto de señales de control que indican a la ALU, a la unidad de memoria y a otros componentes qué operación deben realizar.

## Execute

La fase de ejecución es donde ocurre el trabajo propiamente dicho. La ALU, la FPU o la unidad vectorial realizan la operación especificada por la instrucción:

- **Operaciones aritméticas:** suma, resta, multiplicación, división.
- **Operaciones lógicas:** AND, OR, NOT, XOR, desplazamientos.
- **Acceso a memoria:** carga (*load*) o almacenamiento (*store*) de datos.
- **Saltos:** cambio de la dirección de ejecución si se cumple una condición.

La duración de la fase de ejecución varía según la operación. Una suma de dos registros puede completarse en un solo ciclo, mientras que una división puede requerir múltiples ciclos. Las operaciones de punto flotante y las operaciones vectoriales SIMD pueden requerir ciclos adicionales, especialmente para operandos grandes.

Si la operación requiere datos que no están en los registros internos, el procesador debe acceder a la memoria a través de la jerarquía de cache. Esta espera puede introducir cientos de ciclos de reloj de retraso, convirtiendo el acceso a la memoria en el cuello de botella más crítico del ciclo.

## Writeback

La fase de escritura (*writeback*) almacena el resultado de la operación en el registro de destino o en la memoria. Esta fase cierra el ciclo y prepara al procesador para la siguiente instrucción.

El resultado puede dirigirse a:

- **Un registro interno:** la forma más rápida de almacenamiento, accesible en un solo ciclo.
- **La memoria principal:** utilizado por instrucciones de almacenamiento (*store*), que escriben datos desde los registros a la memoria.
- **Ningún lugar:** algunas instrucciones como las de salto solo modifican el Program Counter y no producen un resultado que deba almacenarse.

En procesadores con ejecución fuera de orden, el writeback debe respetar el orden original de las instrucciones del programa, incluso si se ejecutaron en un orden diferente. Esto garantiza que los resultados finales sean idénticos a los de una ejecución secuencial.

## Tubería (Pipeline)

El *pipelining* es una técnica que permite superponer múltiples instrucciones en diferentes fases de ejecución. Sin pipeline, una instrucción debe completar todas sus fases antes de que la siguiente pueda comenzar. Con pipeline, cada fase puede estar procesando una instrucción diferente simultáneamente.

Un pipeline clásico de cinco etapas permite tener hasta cinco instrucciones en proceso al mismo tiempo:

```
Ciclo:        1    2    3    4    5    6    7
Instrucción 1: Fetch Dec  Exec Wb
Instrucción 2:      Fetch Dec  Exec Wb
Instrucción 3:           Fetch Dec  Exec Wb
Instrucción 4:                Fetch Dec  Exec Wb
```

Sin pipeline, cada instrucción tarda 4 ciclos y se completan 4 instrucciones en 16 ciclos. Con pipeline, las primeras 4 instrucciones se completan en 7 ciclos, casi duplicando el rendimiento.

Los procesadores modernos utilizan pipelines mucho más profundos —de 14 a 20 etapas— lo que permite mayor paralelismo pero también introduce mayor penalización cuando el pipeline debe vaciarse (*flush*) debido a una predicción incorrecta o una dependencia de datos.

## Ejecución fuera de orden

La ejecución fuera de orden (*out-of-order execution*) es una técnica que permite al procesador reordenar las instrucciones para maximizar el uso de los recursos del hardware. Si una instrucción está bloqueada esperando datos de la memoria, el procesador puede ejecutar instrucciones posteriores que no tengan esa dependencia.

El *reorder buffer* (ROB) es el componente encargado de mantener el orden original de las instrucciones. Aunque se ejecuten en un orden diferente, los resultados se escriben en el orden correcto, garantizando la semántica del programa.

Esta técnica es fundamental para el rendimiento de los procesadores modernos. Permite aprovechar los ciclos de reloj que de otra forma se desperdiciarían esperando a que los datos estén disponibles.

> La ejecución fuera de orden requiere un hardware significativamente más complejo que la ejecución en orden. El procesador debe ser capaz de analizar múltiples instrucciones simultáneamente, identificar dependencias y reordenar las operaciones sin alterar el resultado final.

## Predicción de saltos

Las instrucciones de salto condicional (if-else, bucles) presentan un desafío para el pipeline: el procesador no sabe qué instrucciones debe cargar después del salto hasta que la condición se haya evaluado. Si espera a que la condición esté lista, el pipeline se vacía y se desperdician ciclos de reloj.

La predicción de saltos resuelve este problema asumiendo que el salto se tomará (o no se tomará) y comenzando a ejecutar las instrucciones del camino predicho. Si la predicción es correcta, no hay pérdida de rendimiento. Si es incorrecta, el pipeline se vacía y se reanuda desde la instrucción correcta.

Los predictores modernos utilizan historiales de ejecución para mejorar la precisión. Los predictores de dos niveles, los predictores de nivel tournament y los predictores basados en neuronas artificiales alcanzan precisiones superiores al 95% en muchas cargas de trabajo.

### Ejemplo simplificado del ciclo completo

Para una instrucción `ADD R1, R2, R3`:

```
Fetch:      Lee la instrucción ADD de la dirección 0x1000
Decode:     Identifica: operación=ADD, fuente1=R2, fuente2=R3, destino=R1
Execute:    ALU calcula R2 + R3
Writeback:  Resultado se escribe en R1
```

En un pipeline de 5 etapas, esta instrucción se superpone con las siguientes, maximizando el uso del hardware.

