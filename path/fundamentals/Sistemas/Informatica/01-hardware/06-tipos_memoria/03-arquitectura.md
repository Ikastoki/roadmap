# Arquitectura de memoria

La arquitectura de un sistema de memoria no se limita al módulo de RAM instalado en la placa base: incluye toda la jerarquía de niveles de caché dentro del procesador, el controlador de memoria integrado y los canales de comunicación entre ambos. La forma en que estos componentes se organizan y conectan tiene un impacto directo en el rendimiento, particularmente en la cantidad de datos que el procesador puede consumir simultáneamente.

La mayoría de los usuarios solo piensan en la memoria en términos de capacidad total ("tengo 16 GB de RAM"), pero la arquitectura subyacente —niveles de caché, configuración de canales y técnicas como interleaving— determina en gran medida si el sistema se comporta de forma fluida o presenta cuellos de botella inesperados.

## Niveles L1, L2, L3: ubicación y función

El procesador moderno incorpora múltiples niveles de caché integrados directamente en el die, cada uno con características distintas de tamaño, latencia y función:

**Caché L1:** es la memoria más rápida y pequeña, situada en cada núcleo del procesador. Típicamente se divide en un banco de instrucciones (L1i) y un banco de datos (L1d), cada uno de 32-64 KB. Con una latencia de 1-2 ciclos de reloj (aproximadamente 0.5-1 ns en procesadores modernos), la L1 es donde el procesador busca primero los datos que necesita.

**Caché L2:** es un nivel intermedio de mayor tamaño (256 KB - 1 MB por núcleo) y latencia ligeramente superior (3-10 ciclos). En las arquitecturas actuales, cada núcleo tiene su propia caché L2 dedicada, lo que evita la contención entre núcleos. La L2 actúa como respaldo de la L1: cuando un dato no se encuentra en L1 (*cache miss*), se busca en L2 antes de recurrir a niveles superiores.

**Caché L3 (*Last Level Cache*, LLC):** es compartida entre todos los núcleos del procesador, con tamaños que van desde 4 MB en procesadores de gama baja hasta más de 128 MB en los más avanzados. La latencia de la L3 es mayor que la de L1 y L2 (10-30 ciclos), pero sigue siendo órdenes de magnitud más rápida que la RAM principal. La L3 actúa como último filtro antes de acceder a la memoria, y su tamaño compartido permite que los núcleos compartan datos entre sí de forma eficiente.

> La tendencia reciente es incrementar significativamente el tamaño de la caché L2. Los procesadores AMD Zen 4 duplican la L2 respecto a Zen 3, y Intel ha implementado cachés L3 de hasta 360 MB en algunos modelos especializados. Este enfoque refleja la importancia crítica de mantener al procesador abastecido de datos sin recurrir a la RAM.

## Single, Dual, Quad Channel

El controlador de memoria del procesador se comunica con los módulos RAM a través de uno o más canales, cada uno de los cuales proporciona un ancho de banda independiente. La configuración de canales determina cuántos datos pueden transferirse simultáneamente entre la RAM y el procesador:

**Single Channel:** un solo canal de comunicación. Con DDR4-3200, el ancho de banda máximo es de 25.6 GB/s. Es la configuración mínima y la que presenta peor rendimiento.

**Dual Channel:** dos canales operando en paralelo, doblando el ancho de banda teórico a 51.2 GB/s con DDR4-3200. Es la configuración estándar en la mayoría de sistemas de escritorio y portátiles actuales. Para aprovechar el dual channel, los módulos deben instalarse en los slots correctos (típicamente A2 y B2, o según indique el manual de la placa base).

**Triple Channel:** utilizado en algunas plataformas de gama alta (como ciertas variantes de Intel LGA 2011), ofrece 76.8 GB/s con DDR4-3200. Su adopción fue limitada y fue reemplazado por quad channel.

**Quad Channel:** cuatro canales simultáneos, alcanzando 102.4 GB/s con DDR4-3200. Es exclusivo de plataformas de servidor y estaciones de trabajo (Intel Xeon, AMD EPYC). Los módulos se instalan en pares o cuartetos para activar los cuatro canales.

> Instalar módulos en la configuración incorrecta puede hacer que el sistema opere en single channel incluso con dos o cuatro módulos instalados. Consultar el manual de la placa base es indispensable para asegurar la correcta activación de canales múltiples.

## El concepto de interleaving

El interleaving es una técnica que distribuye las direcciones de memoria consecutivas entre los diferentes canales o bancos de memoria, permitiendo que las solicitudes de acceso se distribuyan de forma equilibrada.

En el interleaving a nivel de canal, las direcciones pares van a un canal y las impares al otro. Esto significa que cuando el procesador accede a direcciones consecutivas (algo muy frecuente gracias a la localidad espacial), ambas solicitudes pueden atenderse simultáneamente en canales diferentes, reduciendo la latencia efectiva.

El interleaving a nivel de banco distribuye las direcciones entre los bancos internos de un mismo canal. Los bancos de memoria pueden operar de forma independiente, de modo que mientras un banco está en ciclo de refresco o preparándose para una nueva operación, otro banco puede atender una solicitud. Esto reduce la penalización por conflictos entre bancos.

La combinación de interleaving a nivel de canal y de banco maximiza el uso del ancho de banda disponible y minimiza los periodos de inactividad del controlador de memoria. En la práctica, la activación correcta del interleaving depende tanto del hardware (controlador de memoria, diseño de la placa base) como de la instalación correcta de los módulos en los slots apropiados.

