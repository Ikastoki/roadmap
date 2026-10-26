# Función de la PSU

La fuente de alimentación (*Power Supply Unit*) es el componente responsable de convertir la corriente alterna (AC) de la red eléctrica en la corriente continua (DC) que necesitan todos los componentes internos de una computadora. Sin una fuente de alimentación fiable y adecuada, ningún componente del sistema puede funcionar de forma estable: fluctuaciones de voltaje, cortocircuitos o insuficiencia de potencia pueden causar desde bloqueos aleatorios hasta daños permanentes en hardware costoso.

La PSU es con frecuencia el componente más subestimado en las configuraciones de ordenador. Los usuarios suelen priorizar el procesador y la tarjeta gráfica, olvidando que ambos dependen completamente de la calidad de la energía que les proporciona la fuente. Una mala fuente de alimentación puede inutilizar componentes de miles de euros y comprometer la disponibilidad de todo el sistema.

## Qué es la PSU

La fuente de alimentación es un dispositivo electrónico que toma la corriente alterna de la red (típicamente 110V o 220V según el país) y la transforma en múltiples voltajes de corriente continua: +3.3V, +5V y +12V, que son los voltajes estándar que utilizan los componentes de un ordenador.

El proceso de conversión implica múltiples etapas: rectificación (conversión de AC a DC), filtrado (eliminación de ruido y fluctuaciones) y regulación (mantención de voltajes estables bajo diferentes cargas). Los componentes internos de la PSU —transformadores, condensadores, mosfets y bobinas— trabajan coordinados para proporcionar energía limpia y estable.

## AC vs. DC

La corriente alterna (AC) es la forma en que la electricidad llega desde la red eléctrica: los electrones oscilan alternativamente en ambas direcciones a una frecuencia de 50 o 60 Hz. La corriente continua (DC) fluye en una sola dirección, como la de una batería.

Los componentes electrónicos de un ordenador —CPU, GPU, memoria, almacenamiento— operan exclusivamente con corriente continua. La PSU es el intermediario que convierte la AC en DC, eliminando la componente alterna y estabilizando el voltaje.

## Voltios (V), Amperios (A), Vatios (W)

Estas tres magnitudes eléctricas son fundamentales para entender el funcionamiento de una fuente de alimentación:

**Voltios (V):** mide la diferencia de potencial eléctrico, es decir, la "presión" que impulsa a los electrones a moverse. Un voltaje de 12V significa que existe una diferencia de potencial de 12 voltios entre los conductores.

**Amperios (A):** mide la intensidad de la corriente eléctrica, es decir, la cantidad de electrones que fluyen por unidad de tiempo. Un componente que consume 10A a 12V está extrayendo una cantidad significativa de energía.

**Vatios (W):** mide la potencia eléctrica, que es el producto del voltaje por la corriente:

```
Potencia (W) = Voltaje (V) x Corriente (A)
```

Una fuente de 650W a 12V puede proporcionar hasta 54A en el riel de +12V. Esta capacidad es la que determina si la fuente puede alimentar componentes de alto consumo como una GPU de gama alta.

## Conversión de corriente

El proceso de conversión de AC a DC no es 100% eficiente: parte de la energía se disipa en forma de calor durante la transformación. La eficiencia de una fuente de alimentación se mide como el porcentaje de energía de entrada que se convierte en energía útil de salida.

Una fuente con 80% de eficiencia a 500W de carga consume 625W de la red eléctrica y entrega 500W al sistema, disipando 125W como calor. Una fuente con 90% de eficiencia en las mismas condiciones consume solo 556W, reduciendo tanto el consumo energético como la generación de calor.

> La eficiencia de la PSU no solo afecta al consumo energético: también influye en la temperatura interna de la fuente y en la vida útil de sus componentes. Una fuente más eficiente genera menos calor, lo que reduce el desgaste de los condensadores y otros componentes.

## Importancia de la estabilidad

La estabilidad del voltaje es tan crítica como la cantidad de potencia disponible. Un voltaje inestable o con demasiado ruido puede causar:

- **Bloqueos aleatorios:** el procesador o la GPU reciben un voltaje fluctuante que provoca errores en las operaciones.
- **Degradación acelerada:** los componentes electrónicos sometidos a voltajes inestables envejecen más rápido.
- **Fallo catastrófico:** un pico de voltaje puede quemar componentes sensibles como la CPU o la placa base.

Una buena fuente de alimentación incluye etapas de filtrado y regulación que mantienen los voltajes dentro de especificaciones estrictas, incluso cuando la carga varía rápidamente (por ejemplo, cuando la GPU pasa de estar inactiva a estar a plena carga en milisegundos).

