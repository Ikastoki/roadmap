# Formatos y estándares de placa base

El *form factor* de una placa base define sus dimensiones, la disposición de los componentes y el tipo de chasis compatible. Elegir el factor de forma adecuado es una de las decisiones más importantes al diseñar un sistema, ya que determina qué componentes pueden instalarse, cómo se distribuirá el calor y qué margen de expansión habrá.

No existe un factor de forma universalmente óptimo: cada formato representa un compromiso entre tamaño, capacidad de expansión y facilidad de mantenimiento. Un servidor de rack priorizará la densidad, mientras que una estación de trabajo de video puede necesitar espacio para múltiples tarjetas gráficas y almacenamiento.

## ATX (Advanced Technology Extended)

ATX es el factor de forma dominante en la computación de escritorio desde su introducción por Intel en 1995. Sus dimensiones estándar son 305 x 244 mm (12 x 9.6 pulgadas), aunque existen variaciones menores entre fabricantes.

El diseño ATX establece una disposición específica de los componentes:

- El zócalo de la CPU se sitúa en la parte superior izquierda, cerca de la ventilación trasera del chasis.
- Los slots DIMM de memoria se ubican a la derecha del zócalo de CPU.
- Los slots PCIe de expansión se distribuyen en la parte inferior.
- Los puertos de almacenamiento SATA y los conectores frontales se sitúan en la parte derecha e inferior.

Esta disposición fue concebida para optimizar el flujo de aire dentro del chasis. Al colocar los componentes que más calor generan (CPU y GPU) en la zona de ventilación trasera, el aire fresco puede entrar por la parte delantera y salir por la trasera sin obstáculos significativos.

ATX es la mejor opción para sistemas de escritorio que requieren múltiples tarjetas de expansión, gran capacidad de memoria y opciones de upgrade. La mayoría de placas base de gama media y alta utilizan este factor de forma.

## Micro-ATX (mATX)

Micro-ATX mantiene la misma anchura que ATX (244 mm) pero reduce la altura a 244 mm, creando un cuadrado. Este formato sacrifica parte de los slots de expansión —típicamente cuatro en lugar de siete— pero conserva la compatibilidad con chasis ATX.

Micro-ATX es una buena opción para sistemas donde el tamaño es una preocupación pero no se quiere renunciar completamente a la expansión. Las placas mATX suelen ser más económicas que sus equivalentes ATX, lo que las convierte en una alternativa popular para sistemas domésticos y oficinas.

> La compatibilidad hacia atrás es una ventaja significativa de Micro-ATX: puede instalarse en chasis ATX sin problemas, ya que los puntos de montaje y los puertos traseros son compatibles.

## Mini-ITX

Mini-ITX, creado por VIA Technologies en 2001, es un factor de forma compacto de 170 x 170 mm. Originalmente diseñado para sistemas embebidos y dispositivos de bajo consumo, se ha convertido en la opción preferida para construcciones compactas de alta gama.

A pesar de su pequeño tamaño, Mini-ITX puede albergar componentes potentes: un zócalo de CPU de gama alta, hasta dos slots DIMM de memoria y un slot PCIe x16 para tarjetas gráficas. Sin embargo, el número de slots de expansión es muy limitado (típicamente uno), y la disposición compacta puede dificultar el montaje y el mantenimiento.

Los chasis Mini-ITX son significativamente más pequeños que los ATX, lo que los hace ideales para HTPCs (*Home Theater PCs*), sistemas de oficina con poco espacio o construcciones temáticas donde la estética es prioritaria.

## E-ATX (Extended ATX)

E-ATX amplía las dimensiones de ATX a 305 x 330 mm (12 x 13 pulgadas), proporcionando espacio adicional para más fases de alimentación, más slots DIMM (típicamente ocho) y más slots PCIe. Este formato está orientado a estaciones de trabajo y servidores que requieren múltiples tarjetas gráficas, grandes volúmenes de memoria o conectividad avanzada.

Los chasis E-ATX son considerablemente más grandes y pesados que los ATX estándar, y las placas base E-ATX son significativamente más caras. Este formato solo tiene sentido para aplicaciones profesionales donde el rendimiento y la capacidad de expansión justifican el coste y el tamaño adicional.

## Compatibilidad física

La compatibilidad entre factores de forma sigue una jerarquía:

- Un chasis ATX puede albergar placas ATX, Micro-ATX e Mini-ITX.
- Un chasis Micro-ATX puede albergar placas Micro-ATX e Mini-ITX.
- Un chasis Mini-ITX solo puede albergar placas Mini-ITX.

Esta compatibilidad se debe a que los factores de forma más pequeños mantienen los mismos puntos de montaje y la misma disposición de puertos traseros que los más grandes, solo que con menos espacio total.

Los puertos traseros (*I/O shield*) también son compatibles: el panel trasero de una placa Mini-ITX tiene el mismo tamaño que el de una placa ATX, ya que ambos estándares definen el mismo recorte para los puertos externos.

