# Concepto de memoria

La memoria en un sistema informático es el conjunto de componentes encargados de retener datos e instrucciones durante el proceso de ejecución. Sin memoria, el procesador no tendría dónde cargar las instrucciones que debe ejecutar ni dónde almacenar los resultados intermedios de sus cálculos. Es, junto con la CPU, uno de los pilares fundamentales sobre los que se construye cualquier computadora.

La distinción entre memoria y almacenamiento es uno de los conceptos más importantes que frecuentemente se confunden. Mientras la memoria opera como un espacio de trabajo temporal y de alta velocidad, el almacenamiento persiste los datos cuando el sistema se apaga. Comprender esta diferencia, junto con las métricas que definen el rendimiento de la memoria —latencia, frecuencia y ancho de banda— es esencial para entender cómo un sistema informático gestiona sus recursos más críticos.

## Memoria vs almacenamiento

La memoria principal (*RAM*, *Random Access Memory*) y el almacenamiento (*SSD*, *HDD*) cumplen funciones complementarias pero fundamentalmente distintas. La memoria es el espacio de trabajo activo del sistema: contiene los datos e instrucciones que la CPU está usando o va a usar en un futuro inmediato. El almacenamiento, por su parte, guarda de forma permanente el sistema operativo, las aplicaciones y los datos del usuario.

Una analogía útil es la de un escritorio y un archivador. La memoria sería la superficie del escritorio, donde se colocan los documentos con los que se está trabajando activamente. El archivador represents el almacenamiento: tiene mayor capacidad pero requiere más tiempo para localizar y recuperar un documento específico.

Esta distinción tiene implicaciones prácticas directas. Un sistema con poca memoria pero mucho almacenamiento se verá obligado a mover datos constantemente entre ambos, degradando el rendimiento. Un sistema con suficiente memoria mantendrá los datos más accesibles para el procesador, reduciendo la dependencia del almacenamiento más lento.

## Volatilidad

La volatilidad es la propiedad que define si una tecnología de memoria conserva o pierde su contenido cuando se corta el suministro eléctrico. La RAM es volátil: al apagar el sistema, todos los datos almacenados en ella desaparecen irreversiblemente. Este diseño no es una limitación sino una elección arquitectónica deliberada: la memoria volátil puede ser significativamente más rápida que la no volátil, ya que no necesita mecanismos complejos para preservar la información.

La no volatilidad, por el contrario, permite que los datos persistan sin energía. Las tecnologías no volátiles incluyen las memorias ROM (*Read-Only Memory*), los SSD basados en NAND Flash y los discos duros magnéticos. Estos componentes constituyen el almacenamiento del sistema, donde se guardan de forma permanente el sistema operativo, las aplicaciones y los archivos del usuario.

> La línea entre memoria volátil y no volátil se ha difuminado en los últimos años. Tecnologías como Intel Optane (3D XPoint) buscaban combinar la velocidad de la RAM con la persistencia del almacenamiento, aunque su adopción fue limitada y finalmente discontinuada.

## Latencia

La latencia de memoria es el tiempo que transcurre desde que se realiza una solicitud de acceso hasta que los datos están disponibles. Es una de las métricas más críticas para el rendimiento del sistema, ya que determina cuánto debe esperar la CPU antes de recibir la información que necesita.

La latencia se mide típicamente en nanosegundos (ns) o ciclos de reloj. Los valores varían enormemente según el nivel de la jerarquía de memoria: la caché L1 de un procesador moderno tiene una latencia de aproximadamente 1 ns, la caché L2 entre 3-10 ns, la caché L3 entre 10-30 ns, y la RAM principal entre 50-100 ns. El almacenamiento SSD NVMe tiene latencias del orden de 10,000-100,000 ns, y los discos duros mecánicos pueden superar los 10,000,000 ns.

Estas diferencias de órdenes de magnitud explican por qué los procesadores invierten tanto esfuerzo en mantener los datos más utilizados en los niveles superiores de la jerarquía de caché: cada ciclo de espera por datos de RAM es un ciclo en el que el procesador no está realizando trabajo útil.

## Frecuencia

La frecuencia de la memoria mide cuántas operaciones de acceso puede realizar por unidad de tiempo, expresada en megahercios (MHz) o gigahercios (GHz). A diferencia de la frecuencia del procesador, que indica cuántos ciclos de reloj ejecuta por segundo, la frecuencia de la memoria indica la tasa a la que se transfieren datos entre la RAM y el controlador de memoria.

Las memorias DDR4 actuales operan típicamente entre 2133 MHz y 3600 MHz, mientras que las DDR5 comienzan en 4800 MHz y alcanzan más de 8000 MHz. Sin embargo, esta frecuencia bruta no se traduce directamente en rendimiento de la misma forma que en un procesador: la arquitectura de la memoria y los timings de acceso también juegan un papel determinante.

## Ancho de banda

El ancho de banda de memoria es la cantidad de datos que se pueden transferir entre la RAM y el controlador de memoria en un período de tiempo dado, medido en gigabytes por segundo (GB/s). Se calcula multiplicando la frecuencia efectiva por el ancho del bus de datos y por el número de canales.

Una memoria DDR4-3200 en configuración dual channel tiene un ancho de banda teórico de:

```
Ancho de banda = 3200 MHz x 8 bytes x 2 canales = 51.2 GB/s
```

Una memoria DDR5-6400 en dual channel alcanza:

```
Ancho de banda = 6400 MHz x 8 bytes x 2 canales = 102.4 GB/s
```

Este ancho de banda es crítico para aplicaciones que manejan grandes volúmenes de datos, como el procesamiento de video, la simulación científica o la inteligencia artificial. Sin embargo, para la mayoría de las tareas cotidianas, la latencia de la memoria tiene un impacto mayor en la percepción de rendimiento que el ancho de banda bruto.

## Jerarquía de memoria

La jerarquía de memoria es la organización de los diferentes niveles de almacenamiento de un sistema, ordenados por velocidad, capacidad y proximidad al procesador. En la cúspide se encuentran los registros del procesador, seguidos de las cachés L1, L2 y L3, la RAM principal y finalmente el almacenamiento de largo plazo.

```
Registros        →  ~0 ns      →  Bytes
Caché L1         →  ~1 ns      →  32-64 KB
Caché L2         →  ~3-10 ns   →  256 KB - 1 MB
Caché L3         →  ~10-30 ns  →  4-64 MB
RAM principal    →  ~50-100 ns →  4-128 GB
SSD NVMe         →  ~10,000 ns →  256 GB - 8 TB
HDD              →  ~10,000,000 ns →  1-20 TB
```

El principio fundamental de la jerarquía es el **principio de localidad**: los programas tienden a acceder repetidamente a los mismos datos (localidad temporal) o a datos cercanos en memoria (localidad espacial). La jerarquía de memoria aprovecha este comportamiento manteniendo los datos más utilizados en los niveles más rápidos y cercanos al procesador, mientras que los datos menos activos se relegan a niveles más lentos pero de mayor capacidad.

Este diseño permite acercar el rendimiento de la memoria al de la caché a un coste significativamente menor. Sin la jerarquía, construir una memoria con la velocidad de la caché L1 y la capacidad de la RAM actual sería económicamente inviable, ya que la memoria SRAM utilizada en las cachés es mucho más costosa y ocupa más espacio物理 que la DRAM de la RAM principal.

