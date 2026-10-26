# Definición de bus

Un bus en informática es un sistema de comunicación que permite la transferencia de datos, direcciones y señales de control entre los distintos componentes de un sistema informático. Desde la perspectiva más abstracta, un bus es simplemente un conjunto de líneas conductoras compartidas por múltiples dispositivos, donde la arbitrariedad de acceso —quién puede usar el bus en cada momento— determina la eficiencia y la complejidad del sistema.

La existencia de los buses es una consecuencia directa de la imposibilidad práctica de conectar cada componente directamente con cada otro componente. Sin buses, una CPU necesitaría conexiones dedicadas a cada módulo de memoria, cada dispositivo de E/S y cada periférico, resultando en un entramado de cables inviable. Los buses proporcionan un camino compartido que simplifican drásticamente la interconexión, aunque introducen un nuevo desafío: la gestión del acceso concurrente al medio compartido.

## Bus de datos, bus de direcciones, bus de control

Un bus de sistema se compone típicamente de tres sub-buses funcionales:

**Bus de datos:** transporta la información propiamente dicha entre los componentes. Su anchura (número de líneas paralelas) determina cuántos bits se pueden transferir simultáneamente. Un bus de datos de 64 bits puede transferir 8 bytes en cada ciclo de reloj, mientras que uno de 32 bits solo 4 bytes. El ancho del bus de datos es uno de los factores determinantes del ancho de banda máximo del sistema.

**Bus de direcciones:** transporta la información que identifica la ubicación de origen o destino de los datos. En un sistema con espacio de direcciones de 64 bits, el bus de direcciones debe ser lo suficientemente ancho para representar todas las direcciones posibles de memoria. Sin embargo, en los sistemas modernos, el bus de direcciones no siempre se implementa como un conjunto físico de líneas dedicadas, sino que se multiplexa temporalmente sobre el mismo cableado que el bus de datos.

**Bus de control:** transporta las señales que coordinan las operaciones del bus: peticiones de lectura/escritura, señales de sincronización, interrupciones y arbitraje de acceso. Es el componente más complejo del bus, ya que define el protocolo que gobierna cómo los dispositivos acceden al medio compartido y cómo se resuelven los conflictos cuando múltiples dispositivos solicitan acceso simultáneamente.

> La separación conceptual en tres sub-buses (datos, direcciones, control) es una simplificación útil para comprender el funcionamiento. En la práctica, los buses modernos multiplexan estas señales sobre el mismo cableado físico, utilizando marcas de tiempo para distinguir si un ciclo transfiere una dirección o datos.

## Ancho del bus y velocidad

El ancho del bus (número de líneas de datos) y la frecuencia del reloj del bus determinan juntos el ancho de banda máximo:

```
Ancho de banda = Ancho del bus (bytes) x Frecuencia del reloj (Hz)
```

Un bus de 32 bits operando a 100 MHz proporciona un ancho de banda de 3,200 MB/s (4 bytes x 100 MHz), mientras que uno de 64 bits a 200 MHz alcanza 12,800 MB/s (8 bytes x 200 MHz).

En los buses modernos como PCI Express, el concepto de ancho del bus se traduce en el número de carriles (*lanes*). Cada carril PCIe es un enlace serial de alta velocidad que transfiere datos en ambas direcciones simultáneamente. Un enlace x16 (16 carriles) de PCIe 4.0 ofrece un ancho de banda teórico de 31.5 GB/s, suficiente para alimentar incluso las GPU más potentes.

La tendencia histórica ha sido la migración de buses paralelos de anchos amplios a buses seriales de alta velocidad con menos carriles. Los buses paralelos, aunque intuitivos, sufren problemas de sincronización (*skew*) cuando las líneas largas no mantienen la misma latencia de propagación, limitando su frecuencia máxima. Los buses seriales, al transmitir un bit a la vez a mucha mayor frecuencia, eliminan estos problemas y permiten anchos de banda superiores con menos líneas físicas.

## Cuellos de botella

Un cuello de botella de bus ocurre cuando un componente necesita transferir datos a una velocidad mayor de la que el bus permite, forzando al componente más rápido a esperar al bus más lento. Los cuellos de botella de bus son una de las causas más comunes de rendimiento subóptimo en sistemas informáticos.

El cuello de botella clásico es el bus de conexión entre la CPU y la memoria principal. Si la CPU puede procesar datos a 100 GB/s pero el bus de memoria solo proporciona 50 GB/s, la mitad del potencial de procesamiento se desperdicia en esperas. Este fue precisamente el problema que motivó la introducción de la caché: al almacenar datos frecuentemente accedidos más cerca de la CPU, se reduce la frecuencia con que el bus de memoria debe ser utilizado.

Los buses de E/S presentan cuellos de botella similares. Un SSD NVMe capaz de 7,000 MB/s conectado a un slot PCIe 3.0 x4 limitado a 3,500 MB/s no podrá alcanzar su velocidad máxima. De la misma forma, una GPU de alta gama instalada en un slot PCIe x8 en lugar de x16 verá reducido su ancho de banda disponible, potencialmente afectando el rendimiento en aplicaciones que requieren transferencias masivas de datos entre la GPU y la memoria del sistema.

> La identificación de cuellos de botella es una de las habilidades más importantes en la optimización de sistemas. Herramientas como CPU-Z, GPU-Z y los monitores de rendimiento del sistema permiten identificar qué componente está operando por debajo de su potencial y dónde se está perdiendo rendimiento.

