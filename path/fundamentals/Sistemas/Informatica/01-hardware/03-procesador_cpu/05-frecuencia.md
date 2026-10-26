# Frecuencia y multiplicador

La frecuencia de reloj es la métrica más visible de un procesador: los fabricantes la utilizan como principal argumento de marketing, y los usuarios la asocian intuitivamente con la velocidad. Sin embargo, la frecuencia por sí sola no determina el rendimiento. Su interacción con el multiplicador, el IPC y las técnicas de gestión energética crea un panorama complejo donde dos procesadores con la misma frecuencia pueden tener rendimientos muy diferentes.

Comprender la frecuencia, el multiplicador y sus conceptos relacionados es esencial para tomar decisiones informadas sobre hardware y para entender cómo los procesadores modernos optimizan su rendimiento en tiempo real.

## Frecuencia

La frecuencia de un procesador mide el número de ciclos de reloj que ejecuta por segundo, expresado en hercios (Hz). Un procesador a 3.5 GHz ejecuta 3.500 millones de ciclos por segundo. Cada ciclo representa la unidad básica de tiempo en la que el procesador puede realizar una operación.

La frecuencia es determinada por un oscilador de cristal en la placa base que genera una señal de referencia. Esta señal se multiplica internamente por el procesador para alcanzar la frecuencia final de operación. La estabilidad de la señal de reloj es crítica: fluctuaciones o *jitter* pueden causar errores en las operaciones del procesador.

La frecuencia máxima de un procesador está limitada por múltiples factores:

- **Velocidad de propagación de la señal:** los transistores necesitan un tiempo mínimo para cambiar de estado. A mayor frecuencia, más rápido deben conmutar.
- **Calor generado:** cada ciclo genera calor. A mayor frecuencia, mayor es la potencia térmica que debe disiparse.
- **Consumo energético:** la potencia consumida crece linealmente con la frecuencia (y cuadráticamente con el voltaje).

## Multiplicador

El multiplicador es el factor que determina la frecuencia final del procesador a partir de la frecuencia de referencia (BCLK, *Base Clock*). La fórmula es simple:

```
Frecuencia final = BCLK x Multiplicador
```

Un BCLK de 100 MHz con un multiplicador de 50 produce una frecuencia de 5.000 MHz (5 GHz). El multiplicador es ajustable en la mayoría de procesadores de gama media y alta, lo que permite al usuario incrementar la frecuencia sin modificar el hardware.

El BCLK típico es 100 MHz en Intel y 100 MHz en AMD. Aunque el BCLK también es ajustable, modificarlo afecta a múltiples componentes del sistema (memoria, buses), lo que puede causar inestabilidad. Por esta razón, el overclock se realiza generalmente ajustando el multiplicador y no el BCLK.

> El multiplicador está bloqueado en algunos procesadores de gama baja, impidiendo el overclock. Los procesadores Intel "K" y "KF" y los AMD Ryzen con sufijo "X" o "X3D" desbloquean el multiplicador para permitir ajustes de frecuencia manuales.

## GHz e IPC

La frecuencia (GHz) y el IPC (*Instructions Per Cycle*) son los dos factores que determinan el rendimiento bruto de un procesador. Su producto:

```
Rendimiento teórico = Frecuencia x IPC
```

proporciona una medida más completa del poder de cálculo que la frecuencia sola. Un procesador a 4 GHz con IPC de 2 ejecuta 8.000 millones de instrucciones por segundo, mientras que uno a 3 GHz con IPC de 4 ejecuta 12.000 millones, a pesar de tener menor frecuencia.

Esta relación explica por qué los procesadores ARM pueden competir con los x86 a frecuencias menores: su diseño RISC les permite alcanzar IPC elevados incluso sin frecuencias de reloj extremas. Del mismo modo, un procesador de servidor con muchos núcleos a frecuencia moderada puede superar a uno de escritorio con menos núcleos pero mayor frecuencia en cargas de trabajo paralelas.

## El reloj (clock)

El reloj es la señal periódica que sincroniza todas las operaciones del procesador. Funciona como un metrónomo digital: cada pulso marca el inicio de un ciclo en el que los componentes del procesador ejecutan sus operaciones.

La generación del reloj es una tarea delicada. El oscilador de cristal de la placa base genera una señal base de alta precisión que se distribuye a todos los componentes del sistema. El procesador toma esta señal y la multiplica internamente para alcanzar su frecuencia de operación.

La calidad del reloj afecta directamente a la estabilidad del sistema. Un reloj con mucho *jitter* (variaciones de frecuencia) puede causar errores aleatorios, especialmente durante operaciones críticas o a frecuencias altas. Los sistemas de gama alta incorporan osciladores de alta precisión y circuitos de filtrado para minimizar el jitter.

## La frecuencia base y el overclocking

La **frecuencia base** es la velocidad a la que opera el procesador cuando todos los núcleos están trabajando a plena carga. Es la frecuencia garantizada por el fabricante y la que se utiliza como referencia para el consumo energético y la generación de calor.

La **frecuencia turbo** (o boost) es la velocidad máxima que un núcleo individual puede alcanzar cuando las condiciones lo permiten: temperatura baja, consumo dentro de los límites y pocas cargas en otros núcleos. Los procesadores modernos utilizan algoritmos de gestión energética que ajustan la frecuencia en tiempo real según la carga de trabajo.

El **overclocking** es el proceso de incrementar la frecuencia del procesador por encima de la especificada por el fabricante. Esto se logra aumentando el multiplicador, el voltaje o ambos. El overclock permite obtener mayor rendimiento sin cambiar de hardware, pero tiene implicaciones importantes:

- **Mayor consumo energético:** la potencia consumida crece con la frecuencia y el voltaje.
- **Mayor generación de calor:** el sistema de refrigeración debe ser capaz de disipar el calor adicional.
- **Reducción de la vida útil:** el uso continuado a frecuencias y voltajes elevados acelera el envejecimiento del silicio.
- **Posible inestabilidad:** si el overclock es excesivo, el procesador puede generar errores o bloquearse.

> El overclock responsable implica incrementos graduales, pruebas de estabilidad prolongadas y monitorización constante de temperaturas y voltajes. Un overclock estable a 5 GHz puede no ser útil si el procesador se calienta excesivamente bajo carga continua.

## Turbo Boost y tecnologías similares

Los fabricantes han desarrollado técnicas para ofrecer mayor rendimiento sin comprometer la eficiencia energética:

**Intel Turbo Boost:** permite al procesador incrementar automáticamente la frecuencia de uno o más núcleos por encima de la frecuencia base cuando las cargas de trabajo lo requieren y las condiciones térmicas y eléctricas lo permiten. Turbo Boost Max 3.0 identifica los núcleos individuales con mejor Silicon y les permite alcanzar frecuencias aún más altas.

**AMD Precision Boost:** AMD utiliza un algoritmo similar que ajusta la frecuencia en incrementos de 25 MHz, maximizando el rendimiento dentro de los límites de potencia (PPT), corriente (TDC) y temperatura (Tjmax). Precision Boost Overdrive (PBO) relaja estos límites para obtener mayor rendimiento a costa de mayor consumo.

**Apple Turbo Boost:** en los chips Apple Silicon, la gestión de frecuencia está estrechamente integrada con el sistema operativo, permitiendo transiciones ultrarrápidas entre estados de baja y alta frecuencia para optimizar tanto la respuesta como la eficiencia energética.

Estas tecnologías representan la evolución del concepto de frecuencia fija: en lugar de operar siempre a la misma velocidad, los procesadores modernos adaptan dinámicamente su frecuencia a las necesidades de la carga de trabajo, maximizando el rendimiento cuando es necesario y minimizando el consumo cuando no lo es.

