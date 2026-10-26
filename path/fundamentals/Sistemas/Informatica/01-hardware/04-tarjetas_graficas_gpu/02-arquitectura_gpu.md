# Arquitectura de una GPU

La arquitectura de una GPU es fundamentalmente distinta a la de una CPU. Mientras que la CPU prioriza la latencia baja y la ejecución secuencial eficiente, la GPU está construida para maximizar el throughput: la cantidad de trabajo completado por unidad de tiempo. Cada componente de la GPU está diseñado para alimentar miles de núcleos de procesamiento que ejecutan operaciones simples en paralelo masivo.

Comprender la arquitectura gráfica es esencial tanto para optimizar el rendimiento de videojuegos y aplicaciones de renderizado como para aprovechar al máximo las capacidades de cómputo general de las GPU en cargas de trabajo de IA y ciencia.

## Núcleos gráficos

Los núcleos gráficos son las unidades de procesamiento básicas dentro de una GPU. A diferencia de los núcleos de una CPU, que son completos y autónomos, los núcleos gráficos son mucho más simples y están diseñados para ejecutar operaciones específicas de rasterización y sombreado.

El número de núcleos gráficos en una GPU moderna puede ser enormemente alto: las GPU de gama alta como la NVIDIA RTX 4090 contienen más de 16.000 núcleos CUDA, mientras que las AMD Radeon RX 7900 XTX superan los 6.100 *stream processors*. Cada uno de estos núcleos puede ejecutar una operación de punto flotante por ciclo de reloj, lo que se traduce en un poder de cálculo teórico de decenas de teraflops.

La clave del rendimiento de la GPU no está solo en el número de núcleos, sino en la eficiencia con que se organizan y gestionan. Los núcleos se agrupan en bloques que comparten memoria y recursos de control, lo que permite coordinar el trabajo de miles de núcleos de forma eficiente.

## CUDA Cores / Stream Processors

**CUDA Cores (NVIDIA):** son los núcleos de procesamiento的基本icos de la arquitectura de NVIDIA. Cada CUDA Core puede ejecutar una operación de punto flotante de precisión simple (FP32) por ciclo de reloj. Las arquitecturas más recientes incluyen también núcleos de precisión doble (FP64), núcleos de tensor (para IA) y núcleos de trazado de rayos (RT Cores).

**Stream Processors (AMD):** son el equivalente funcional de los CUDA Cores en las GPU de AMD. Cada stream processor ejecuta una operación SIMD de 5 SIMD lanes, lo que significa que cada *CU* (Compute Unit) de AMD ejecuta 64 operaciones de punto flotante por ciclo de reloj, aunque técnicamente tiene 64 stream processors visibles para el programador.

La diferencia en la nomenclatura no es solo cosmética: refleja diferencias en la arquitectura interna que afectan al rendimiento en diferentes tipos de cargas de trabajo.

## SM / CU

**Streaming Multiprocessor (SM) en NVIDIA:** cada SM es un bloque que contiene múltiples CUDA Cores, registros compartidos, caches y unidades de carga/almacenamiento. Un SM de la arquitectura Ada Lovelice (RTX 40) contiene 128 CUDA Cores, 4 unidades de tensor y 32 cores de trazado de rayos. Las GPU de gama alta pueden tener más de 100 SM.

**Compute Unit (CU) en AMD:** cada CU es el equivalente funcional del SM de NVIDIA. Contiene 64 *stream processors*, unidades de textura y una caché compartida. Las GPU AMD modernas pueden tener hasta 96 CU.

La organización en SM/CU es crítica para el rendimiento porque los recursos dentro de un bloque se comparten entre los núcleos que lo forman. Si un programa requiere más registros de los disponibles en un SM, el compilador debe dividir el trabajo en menos bloques, reduciendo el paralelismo efectivo.

## Rasterización y shaders

La rasterización es el proceso de convertir la geometría 3D (definida por vértices y polígonos) en una imagen 2D compuesta por píxeles. Este proceso es el corazón del renderizado gráfico y es ejecutado por unidades especializadas dentro de la GPU:

**Vertex Shaders:** transforman los vértices del modelo 3D al espacio de pantalla, aplicando rotaciones, escalados y proyecciones perspectivas.

**Fragment Shaders (Pixel Shaders):** determinan el color de cada píxel en función de la textura, la iluminación, las sombras y otros efectos visuales. Los fragment shaders son los que más tiempo de cómputo consumen en los videojuegos modernos.

**Geometry Shaders:** pueden crear o eliminar geometría de forma dinámica, útil para efectos como el *tessellation* o la generación de partículas.

> Las GPU modernas utilizan un pipeline de renderizado unificado donde los mismos núcleos pueden ejecutar diferentes tipos de shaders, permitiendo una mayor eficiencia en la utilización de los recursos.

## ROPs y TMUs

**ROPs (*Render Output Units*):** son las unidades encargadas de las operaciones finales del pipeline gráfico: composición de colores, antialiasing, blending y escritura de los píxeles finales en el framebuffer. El número de ROPs determina la tasa máxima de píxeles que la GPU puede producir por ciclo de reloj.

**TMUs (*Texture Mapping Units*):** son las unidades encargadas de aplicar texturas a las superficies 3D. Realizan filtrado de texturas, mipmap y compresión, operaciones que requieren acceso a la memoria de texturas de la GPU. Un mayor número de TMUs permite renderizar escenas con texturas de mayor resolución sin penalización.

