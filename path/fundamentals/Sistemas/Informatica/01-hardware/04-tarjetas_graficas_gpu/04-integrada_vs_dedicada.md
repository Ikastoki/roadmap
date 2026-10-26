# GPU integrada vs dedicada

La elección entre una GPU integrada y una dedicada es una de las decisiones más importantes al configurar un sistema informático. Cada tipo de GPU representa un compromiso diferente entre rendimiento gráfico, coste, consumo energético y espacio físico. No existe una opción universalmente superior: la elección depende del uso previsto del sistema, del presupuesto y de las prioridades del usuario.

## GPU integrada

Una GPU integrada (*iGPU*) es una unidad gráfica incorporada directamente en el chip del procesador. Comparte la memoria RAM del sistema con la CPU y其他 componentes, lo que elimina la necesidad de una tarjeta gráfica dedicada y su propia memoria VRAM.

### Ventajas

- **Bajo coste:** al estar integrada en la CPU, no requiere una tarjeta adicional, lo que reduce el coste total del sistema.
- **Bajo consumo energético:** las GPU integradas consumen significativamente menos que las dedicadas, ya que operan a voltajes y frecuencias más bajos.
- **Menor generación de calor:** no requieren sistemas de refrigeración dedicados, lo que simplifica el diseño del chasis.
- **Compacta:** ideales para sistemas Small Form Factor (SFF) y portátiles donde el espacio es limitado.
- **Salida de video múltiple:** suelen incluir múltiples puertos de salida (HDMI, DisplayPort) integrados en la placa base.

### Desventajas

- **Rendimiento gráfico limitado:** al compartir la memoria con la CPU y tener menos núcleos gráficos, su rendimiento es significativamente inferior al de una GPU dedicada.
- **Memoria compartida:** el ancho de banda de la RAM del sistema es mucho menor que el de una VRAM dedicada, lo que limita la resolución y el nivel de detalle que pueden manejar.
- **No apta para gaming exigente:** las GPU integradas actuales pueden ejecutar juegos populares a 1080p con ajustes bajos-medios, pero no están diseñadas para gaming de alta gama.

### Casos de uso

- Sistemas de oficina y productividad.
- HTPCs (*Home Theater PCs*) para reproducción de multimedia.
- Portátiles ligeros y ultrabooks.
- Sistemas embebidos y dispositivos IoT.
- Gaming casual a 1080p con ajustes moderados.

## GPU dedicada

Una GPU dedicada (*dGPU*) es una tarjeta gráfica independiente que se inserta en un slot PCIe de la placa base. Incorpora su propio chip GPU, su propia VRAM y su propio sistema de refrigeración, proporcionando rendimiento gráfico significativamente superior al de las GPU integradas.

### Ventajas

- **Alto rendimiento gráfico:** los miles de núcleos gráficos y la VRAM dedicada permiten ejecutar juegos y aplicaciones de renderizado a resoluciones y taxas de refresco elevadas.
- **VRAM dedicada:** el ancho de banda de la VRAM es mucho mayor que el de la RAM del sistema, lo que permite cargar texturas de alta resolución sin penalización.
- **Compatibilidad con cómputo general:** las GPU dedicadas son la plataforma de referencia para el entrenamiento de modelos de IA y simulaciones científicas.
- **Escalabilidad:** los usuarios pueden elegir entre múltiples gamas de rendimiento según su presupuesto y necesidades.

### Desventajas

- **Mayor coste:** una GPU dedicada de gama media puede costar más que el resto del sistema combinado.
- **Mayor consumo energético:** las GPU dedicadas de gama alta pueden consumir más de 300W bajo carga, lo que requiere una fuente de alimentación potente y un sistema de refrigeración eficiente.
- **Mayor generación de calor y ruido:** los sistemas de refrigeración de las GPU dedicadas generan calor y ruido significativos, especialmente bajo carga.
- **Ocupa espacio:** la tarjeta gráfica puede medir hasta 330 mm de largo y ocupar 3 slots de expansión, lo que limita la compatibilidad con ciertos chasis.

### Casos de uso

- Gaming de alta gama a 1080p, 1440p y 4K.
- Edición y renderizado de video profesional.
- Diseño gráfico 3D y CAD.
- Entrenamiento de modelos de inteligencia artificial.
- Simulaciones científicas y de ingeniería.

## Comparativa de rendimiento

La diferencia de rendimiento entre GPU integradas y dedicadas es sustancial. Una GPU integrada como la Intel Iris Xe puede alcanzar 2-4 TFLOPS de rendimiento gráfico, mientras que una GPU dedicada de gama alta como la NVIDIA RTX 4090 supera los 80 TFLOPS.

Sin embargo, esta comparación no es del todo justa: las GPU integradas modernas como las Apple M3 o los AMD Ryzen con Radeon 780M han alcanzado niveles de rendimiento que hace apenas unos años eran exclusivos de GPU dedicadas de gama media.

> La brecha entre GPU integradas y dedicadas se está reduciendo, pero sigue siendo significativa para cargas de trabajo exigentes. La elección entre ambas debe basarse en las necesidades reales del usuario, no en especificaciones teóricas.

## Tendencias actuales

La industria está evolucionando hacia una mayor convergencia entre GPU integradas y dedicadas. Los chips Apple Silicon demuestran que es posible lograr rendimiento gráfico competitivo sin una GPU dedicada, mientras que las GPU integradas de AMD y Intel siguen mejorando generación tras generación.

Al mismo tiempo, las GPU dedicadas se están diversificando en variantes especializadas: tarjetas de trabajo profesional (NVIDIA RTX Ada), aceleradores de IA (NVIDIA H100, AMD MI300) y tarjetas compactas para sistemas SFF.

