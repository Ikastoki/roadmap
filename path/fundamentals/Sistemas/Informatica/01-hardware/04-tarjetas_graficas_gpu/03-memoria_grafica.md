# Memoria gráfica

La memoria gráfica (*VRAM*, *Video RAM*) es uno de los factores más determinantes del rendimiento de una GPU. A diferencia de la memoria del sistema (RAM), que está optimizada para baja latencia, la VRAM está diseñada para proporcionar un ancho de banda masivo capaz de alimentar a miles de núcleos gráficos simultáneamente. La cantidad, tipo y velocidad de la VRAM determinan qué resoluciones, niveles de detalle y taxas de refresco puede manejar una GPU, así como su capacidad para ejecutar cargas de trabajo de cómputo general.

## VRAM

La VRAM es una memoria dedicada que reside físicamente en la tarjeta gráfica, junto al chip GPU. A diferencia de la RAM del sistema, que es compartida entre todos los componentes, la VRAM es exclusiva de la GPU y está optimizada para operaciones de lectura/escritura masivas y concurrentes.

Las modernas GDDR6 y GDDR6X son las tecnologías de VRAM más extendidas en tarjetas de consumo. GDDR6X, utilizada por NVIDIA en las series RTX 30 y 40, alcanza velocidades de transferencia de hasta 21 Gbps por pin, proporcionando un ancho de banda total que puede superar los 1 TB/s en configuraciones de 384 bits.

AMD ha adoptado una tecnología alternativa llamada *Infinity Cache*, una caché de gran tamaño integrada en el chip GPU que reduce la dependencia de la VRAM externa, permitiendo anchos de banda efectivos superiores con buses de memoria más estrechos.

## Ancho de banda

El ancho de banda de la VRAM mide cuántos datos se pueden transferir entre la memoria y la GPU por unidad de tiempo. Se expresa en gigabytes por segundo (GB/s) y es uno de los factores más críticos del rendimiento gráfico.

Una GPU de gama alta como la RTX 4090 alcanza un ancho de banda de más de 1.000 GB/s, lo que permite cargar texturas de alta resolución, geometría compleja y buffers de renderizado a velocidades que mantienen los frames por segundo por encima de 60 en resoluciones 4K.

El ancho de banda se calcula multiplicando la velocidad de transferencia por el ancho del bus de memoria:

```
Ancho de banda = Velocidad de transferencia x Ancho del bus / 8
```

Una GPU con GDDR6 a 20 Gbps y un bus de 256 bits proporciona un ancho de banda de 640 GB/s.

## Tipos de memoria gráfica

A lo largo de la historia de las GPU, se han utilizado múltiples tecnologías de memoria gráfica:

**GDDR (Graphics DDR):** la familia de memorias más extendida. GDDR5, GDDR6 y GDDR6X proporcionan excelente relación entre coste y rendimiento, y son la opción estándar para la mayoría de tarjetas gráficas de consumo.

**HBM (*High Bandwidth Memory*):** utiliza un diseño 3D donde múltiples capas de memoria DRAM se apilan verticalmente y se conectan a la GPU mediante *through-silicon vias* (TSVs). HBM2e y HBM3 alcanzan anchos de banda superiores a 3 TB/s, pero su coste es significativamente mayor. Se utiliza en tarjetas de gama profesional (NVIDIA A100, AMD MI250) y en algunas tarjetas de gama alta de consumo (AMD Radeon VII).

**DDR4/DDR5 en GPU integradas:** las GPU integradas en procesadores comparten la memoria RAM del sistema, lo que reduce el coste pero también el rendimiento gráfico, ya que el ancho de banda de la RAM del sistema es mucho menor que el de una VRAM dedicada.

## Buses de memoria

El ancho del bus de memoria determina cuántos datos se pueden transferir simultáneamente entre la VRAM y la GPU. Un bus más ancho proporciona mayor ancho de banda, pero también aumenta el coste y el consumo energético de la tarjeta.

| Segmento | Ancho de bus típico | Ejemplo |
|---|---|---|
| Gama baja | 64-128 bits | GTX 1650 |
| Gama media | 192-256 bits | RTX 4060 Ti |
| Gama alta | 256-384 bits | RTX 4080 |
| Extrema | 384-512 bits | RTX 4090 |

El ancho del bus interactúa con la velocidad de la memoria para determinar el ancho de banda total. Una GPU con un bus de 192 bits y GDDR6 a 18 Gbps alcanza un ancho de banda de 432 GB/s, mientras que una con bus de 384 bits y GDDR6X a 21 Gbps alcanza más de 1.000 GB/s.

## Velocidad y ancho de banda

La velocidad de la VRAM ha crecido drásticamente en las últimas generaciones. La GDDR5, que fue el estándar durante años, alcanzaba velocidades de hasta 8 Gbps por pin. La GDDR6 duplicó esa cifra a 16-20 Gbps, y la GDDR6X la llevó a 21-24 Gbps.

Sin embargo, el ancho de banda por sí solo no determina el rendimiento real. La latencia de la memoria, la eficiencia de los controladores y la capacidad de la caché de la GPU también influyen en cómo se aprovecha ese ancho de banda.

## Interfaz de conexión

La interfaz de conexión entre la GPU y la placa base es el bus PCIe, que proporciona el canal por el cual los datos se transfieren entre la memoria del sistema y la VRAM.

### La importancia del slot PCIe x16

El slot PCIe x16 es la interfaz estándar para tarjetas gráficas de escritorio. En PCIe 4.0, un slot x16 proporciona un ancho de banda de 32 GB/s (16 GB/s en cada dirección). En PCIe 5.0, esta cifra se duplica a 64 GB/s.

Para la mayoría de las aplicaciones de gaming, PCIe 4.0 x16 proporciona suficiente ancho de banda para que la GPU y la CPU se comuniquen sin cuellos de botella. Sin embargo, las cargas de trabajo de cómputo general que transfieren grandes volúmenes de datos entre la CPU y la GPU (como el entrenamiento de modelos de IA) pueden beneficiarse del ancho adicional de PCIe 5.0.

### Ancho de banda y carriles (lanes)

El número de carriles PCIe disponibles varía según la CPU y la placa base. Las CPU de gama alta suelen proporcionar 16-20 carriles PCIe dedicados a la GPU, mientras que las de gama baja pueden limitar el slot a x8 o incluso x4, reduciendo el ancho de banda disponible.

> En la mayoría de los escenarios de gaming, la diferencia de rendimiento entre PCIe 3.0 x16 y PCIe 4.0 x16 es inferior al 5%. El cuello de botella real está en la capacidad de procesamiento de la GPU, no en el ancho de banda del bus.

