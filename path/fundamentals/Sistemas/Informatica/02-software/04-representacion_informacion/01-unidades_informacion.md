# Unidades de información

Toda la información que una computadora procesa, almacena y transmite se representa fundamentalmente como una secuencia de bits: la unidad mínima de información digital. A partir de esta pieza elemental, se construyen todas las representaciones de datos que conocemos: números, texto, imágenes, audio y vídeo. Comprender cómo se estructuran y combinan estos bits es la base para entender cómo funciona cualquier sistema informático.

La elección de las unidades de información y su relación con las capacidades de hardware define los límites de lo que un sistema puede representar y procesar. Un byte puede representar 256 caracteres diferentes; un entero de 32 bits puede representar números de hasta 4 mil millones; una imagen de 24 bits por píxel puede representar más de 16 millones de colores. Estas capacidades no son arbitrarias: son el resultado de decisiones de diseño históricas que equilibran funcionalidad, eficiencia y compatibilidad.

## Qué es un bit

Un bit (*binary digit*) es la unidad más pequeña de información en un sistema digital. Un bit puede tener únicamente dos valores posibles: 0 o 1. Esta dualidad binaria no es una limitación sino una elección deliberada: los circuitos electrónicos son inherentemente binarios, ya que los transistores operan como interruptores que pueden estar encendidos (1) o apagados (0).

El poder del bit reside en su simplicidad y en la posibilidad de combinar múltiples bits para representar información compleja. Un solo bit puede representar dos estados (verdadero/falso, encendido/apagado, sí/no), pero una secuencia de 8 bits puede representar 256 estados diferentes, y una de 64 bits puede representar más de 18 quintillones.

La representación física de un bit varía según el medio de almacenamiento: voltaje alto/bajo en circuitos electrónicos, polaridad norte/sur en discos magnéticos,反射/refracción en discos ópticos, carga/no carga en celdas de memoria DRAM. Independientemente de la implementación física, el bit siempre representa lo mismo: una distinción binaria entre dos estados.

## Qué es un byte

Un byte es un conjunto de 8 bits, y constituye la unidad básica para representar caracteres y datos en la mayoría de los sistemas informáticos modernos. Con 8 bits, un byte puede representar 2^8 = 256 valores diferentes (del 0 al 255).

La elección de 8 bits como tamaño estándar del byte tiene raíces históricas: los primeros microprocesadores de 8 bits (Intel 8080, MOS 6502) procesaban datos en grupos de 8 bits, y esta convención se perpetuó a lo largo de décadas de evolución tecnológica. Aunque arquitecturalmente un "byte" podría haber sido de cualquier tamaño, el estándar de 8 bits se consolidó por su capacidad para representar el conjunto de caracteres ASCII (128 caracteres) con margen sobrante, y por su eficiencia en términos de hardware.

En la práctica, el byte es la unidad de dirección de memoria más pequeña que puede accederse de forma independiente en la mayoría de las arquitecturas. Aunque la memoria se organiza internamente en palabras más grandes (64 bits en procesadores modernos), el byte sigue siendo la unidad de referencia para el software.

## Múltiplos del byte

Los múltiplos del byte expresan cantidades progresivamente mayores de información:

| Unidad | Símbolo | Equivalencia | Uso típico |
|--------|---------|-------------|------------|
| Byte | B | 8 bits | Carácter individual |
| Kilobyte | KB | 1,024 bytes | Archivo de texto pequeño |
| Megabyte | MB | 1,024 KB | Foto, canción MP3 |
| Gigabyte | GB | 1,024 MB | Película HD, RAM típica |
| Terabyte | TB | 1,024 GB | Disco duro de escritorio |
| Petabyte | PB | 1,024 TB | Centro de datos |
| Exabyte | EB | 1,024 PB | Tráfico de Internet global |

Existe una distinción importante entre el sistema decimal (prefijos del SI) y el sistema binario:

**Sistema decimal (SI):** 1 KB = 1,000 bytes, 1 MB = 1,000 KB. Utilizado por los fabricantes de almacenamiento y redes.

**Sistema binario (IEC):** 1 KiB = 1,024 bytes, 1 MiB = 1,024 KiB. Utilizado por los sistemas operativos para expresar tamaños de archivos y memoria.

Esta discrepancia explica por qué un disco duro de 1 TB anunciado por el fabricante aparece como 931 GB en el sistema operativo: el fabricante usa el sistema decimal (1 TB = 10^12 bytes), mientras que el sistema operativo usa el binario (1 TB = 2^40 bytes = 1,099,511,627,776 bytes, que en GB binarios son ~931).

> La confusión entre sistemas decimal y binario ha generado disputas legales y frustración entre usuarios. La IEC introdujo los prefijos KiB, MiB, GiB para eliminar la ambigüedad, pero su adopción ha sido lenta y la confusión persiste.

## Por qué 8 bits

La elección de 8 bits como tamaño estándar del byte no es arbitraria sino el resultado de una convergencia de factores históricos y prácticos:

**Herencia de los primeros microprocesadores:** los primeros microprocesadores de uso general (Intel 8080, 1974) procesaban datos en grupos de 8 bits. Los sistemas construidos sobre estos procesadores adoptaron el byte de 8 bits como unidad fundamental, y esta convención se perpetuó en las generaciones sucesivas.

**Capacidad de representación:** 8 bits permiten representar 256 valores diferentes, suficiente para el conjunto de caracteres ASCII (128 caracteres) con margen para caracteres extendidos, números del 0 al 255 y cualquier otro dato que requiera un rango moderado de valores.

**Eficiencia en hardware:** los buses de datos de 8, 16, 32 y 64 bits son múltiplos del byte, lo que simplifica el diseño de los circuitos de procesamiento y memoria. Un bus de 32 bits transfiere 4 bytes por ciclo; uno de 64 bits, 8 bytes.

