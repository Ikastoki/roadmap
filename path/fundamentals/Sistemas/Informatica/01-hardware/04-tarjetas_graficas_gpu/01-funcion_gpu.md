# Función de la GPU

La unidad de procesamiento gráfico (*Graphics Processing Unit*) es un procesador especializado diseñado para ejecutar operaciones de renderizado y procesamiento paralelo de datos a velocidades extraordinariamente altas. Aunque su nombre original hacía referencia exclusivamente a los gráficos, las GPU modernas han trascendido su propósito inicial para convertirse en aceleradores de cómputo general capaces de procesar inteligencia artificial, simulaciones científicas, minería de criptomonedas y prácticamente cualquier carga de trabajo que pueda paralelizarse.

La distinción entre CPU y GPU no es solo una cuestión de rendimiento gráfico: refleja una diferencia fundamental en la filosofía de diseño. Mientras que la CPU está optimizada para ejecutar pocas tareas complejas de forma secuencial con la menor latencia posible, la GPU está diseñada para ejecutar miles de tareas simples simultáneamente, sacrificando la latencia individual por el rendimiento agregado.

## Evolución de la GPU

Las primeras tarjetas gráficas eran dispositivos pasivos que simplemente transferían datos de la memoria al monitor sin procesamiento significativo. La revolution comenzó en 1999 cuando NVIDIA introdujo la GeForce 256, el primer chip que la empresa denominó "GPU" por su capacidad de realizar cálculos de transformación y iluminación de vértices en hardware.

Desde entonces, la evolución ha sido vertiginosa:

- **Década de 2000:** las GPU adquirieron capacidades de programación con *vertex shaders* y *pixel shaders*, permitiendo a los desarrolladores crear efectos visuales personalizados.
- **Década de 2010:** la aparición de CUDA (NVIDIA, 2007) y OpenCL abrió las GPU al cómputo general, permitiendo utilizarlas para tareas no gráficas como simulaciones físicas y aprendizaje automático.
- **Década de 2020:** las GPU se han convertido en la plataforma dominante para el entrenamiento de modelos de inteligencia artificial, con chips como los NVIDIA H100 y AMD MI300 diseñados específicamente para cómputo de IA.

> La evolución de las GPU ha sido tan rápida como la de las CPU. En las últimas dos décadas, el rendimiento gráfico ha crecido a un ritmo superior al 50% anual, superando con creces la tasa de mejora de las CPU.

## Diferencias entre CPU y GPU

La diferencia fundamental entre CPU y GPU radica en su arquitectura interna:

| Característica | CPU | GPU |
|---|---|---|
| Núcleos | 4-64 | Miles (1000-18.000+) |
| Tarea por núcleo | Compleja, secuencial | Simple, paralela |
| Latencia individual | Muy baja | Mayor |
| Rendimiento agregado | Moderado | Enorme |
| Memoria | Compartida, baja latencia | Dedicada, alto ancho de banda |
| Reloj | 3-6 GHz | 1-3 GHz |

Una CPU con 16 núcleos puede ejecutar 16 hilos simultáneamente. Una GPU con 10.000 núcleos puede ejecutar 10.000 operaciones simples al mismo tiempo, pero cada operación individual es mucho menos potente que la de una CPU.

Esta diferencia se explica por el tipo de cargas de trabajo para el que fueron diseñadas. Las GPU se concibieron originalmente para rasterizar millones de píxeles de pantalla simultáneamente: cada píxel requiere operaciones matemáticas simples (multiplicaciones de matrices, interpolaciones), pero hay millones de ellos que procesar. Una CPU, por el contrario, está diseñada para ejecutar algoritmos complejos con muchas dependencias secuenciales, como la lógica de un programa de office o la gestión de un sistema operativo.

## Procesamiento paralelo

El paralelismo masivo es la característica definitoria de las GPU. Mientras que la CPU utiliza técnicas como el pipelining y la ejecución fuera de orden para extraer paralelismo de código inherentemente secuencial, la GPU parte de un modelo explícitamente paralelo: miles de núcleos ejecutan la misma instrucción sobre datos diferentes simultáneamente.

Este modelo se conoce como SIMD (*Single Instruction, Multiple Data*) en su forma más pura. Un *kernel* de GPU —una función que se ejecuta en los núcleos gráficos— puede procesar millones de elementos de un array con la misma operación, cada núcleo trabajando sobre un elemento diferente.

La programación de GPU requiere un cambio de mentalidad significativo. Los algoritmos secuenciales que funcionan eficientemente en CPU pueden ser ineficientes en GPU si no se paralelizan adecuadamente. Del mismo modo, algoritmos que parecen ineficientes en CPU pueden ejecutarse órdenes de magnitud más rápido en GPU aprovechando el paralelismo masivo.

## Más allá del videojuego

Las GPU modernas son plataformas de cómputo general que trascienden su propósito gráfico original.

### Cómputo General (GPGPU)

El *General-Purpose computing on Graphics Processing Units* (GPGPU) utiliza las GPU para ejecutar algoritmos no gráficos. Frameworks como CUDA (NVIDIA), ROCm (AMD) y OpenCL permiten a los programadores escribir código que se ejecuta directamente en los núcleos de la GPU, aprovechando su paralelismo masivo para acelerar cálculos científicos, análisis de datos y procesamiento de señales.

### Renderizado

Las GPU siguen siendo la plataforma dominante para el renderizado de gráficos 3D en tiempo real. Los videojuegos, la realidad virtual, la visualización arquitectónica y los efectos visuales de cine dependen de las GPU para calcular la iluminación, las sombras, las reflexiones y las texturas de escenas complejas a decenas de frames por segundo.

### Minería

Las GPU revolucionaron la minería de criptomonedas al permitir el cálculo masivo de hashes necesarios para la prueba de trabajo de Bitcoin y otras criptomonedas. Aunque los dispositivos ASIC han reemplazado a las GPU en la minería de Bitcoin, las GPU siguen siendo relevantes para criptomonedas como Ethereum Classic y Ravencoin.

### Inteligencia Artificial y Machine Learning

Las GPU se han convertido en la plataforma de referencia para el entrenamiento de modelos de inteligencia artificial. Las operaciones de multiplicación de matrices y productos escalares-vectoriales que son la base del aprendizaje profundo se ejecutan eficientemente en los núcleos de la GPU, reduciendo los tiempos de entrenamiento de semanas a horas.

> La demanda de GPU para entrenamiento de IA ha creado una escasez global de chips gráficos, impulsando los precios y dificultando el acceso a hardware de alta gama para gamers y profesionales de renderizado.

