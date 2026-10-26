# Cálculo de potencia

Elegir una fuente de alimentación con la potencia adecuada es una de las decisiones más importantes al configurar un sistema informático. Una fuente con potencia insuficiente causará bloqueos, reinicios y posibles daños en los componentes, mientras que una excesivamente potente desperdicia dinero y opera fuera de su rango de máxima eficiencia. El cálculo correcto de la potencia requerida implica sumar el consumo de todos los componentes y añadir un margen de seguridad para fluctuaciones y futuras ampliaciones.

## Cómo estimar la potencia requerida por el sistema

El proceso de estimación comienza por identificar el consumo de cada componente y sumarlo:

**CPU:** el consumo de la CPU varía enormemente según el modelo y la carga de trabajo. Un AMD Ryzen 5 7600 tiene un TDP de 65W, mientras que un Intel Core i9-14900K puede consumir más de 250W bajo carga. El TDP no representa el consumo máximo real: los procesadores modernos pueden superar el TDP en picos breves de boost.

**GPU:** la GPU es con frecuencia el componente que más energía consume. El TDP de la GPU (denominado TBP, *Total Board Power*) varía desde 75W en tarjetas de gama baja hasta más de 450W en las de gama alta. Es fundamental consultar las especificaciones del fabricante para obtener el consumo real.

**Memoria RAM:** cada módulo DDR4 consume entre 3-5W, y DDR5 entre 4-7W. Con 4 módulos, el consumo total de memoria raramente supera los 25W.

**Almacenamiento:** un disco duro mecánico consume entre 5-10W, y un SSD SATA entre 2-5W. Los SSD NVMe pueden consumir hasta 8-10W bajo carga.

**Placa base y otros componentes:** la placa base, los ventiladores, la iluminación RGB y otros dispositivos periféricos suelen consumir entre 30-50W en total.

Una vez sumados todos los componentes, se obtiene el consumo total del sistema. Por ejemplo:

```
CPU: 125W (TDP)
GPU: 250W (TBP)
RAM: 15W (2 módulos)
SSD NVMe: 8W
Placa base + ventiladores: 40W
Total: 438W
```

## Margen de seguridad

El margen de seguridad es la diferencia entre el consumo real del sistema y la potencia máxima de la fuente. Un margen del 20-30% es recomendable para garantizar que la fuente opere en su rango de máxima eficiencia y tenga capacidad para absorber picos de consumo.

En el ejemplo anterior, con un consumo de 438W, una fuente de 550W proporcionaría un margen del 25%, mientras que una de 650W ofrecería un margen del 47%. Para sistemas con GPU de alta gama que pueden experimentar picos de consumo superiores al TBP nominal, un margen del 30-40% es prudente.

> Un margen excesivo no es beneficioso: una fuente de 1000W operando al 40% de carga (400W) puede ser menos eficiente que una de 650W operando al 70% de carga (455W). Elegir una fuente dimensionada adecuadamente para la carga real del sistema optimiza tanto la eficiencia como el coste.

## Ruido eléctrico

El ruido eléctrico (*ripple*) son fluctuaciones de voltaje de alta frecuencia que acompañan a la corriente continua de salida. Un exceso de ruido puede causar inestabilidad en los componentes sensibles, especialmente la CPU y la memoria RAM.

La especificación ATX establece un límite máximo de 120mV de ripple en el riel de +12V y 50mV en los rieles de +5V y +3.3V. Las fuentes de calidad mantienen el ripple muy por debajo de estos límites, mientras que las fuentes baratas pueden acercarse o incluso superar las especificaciones.

El ripple se reduce mediante condensadores de filtrado de alta calidad y diseños de circuito que minimizan las interferencias electromagnéticas. Las reseñas independientes que miden el ripple son una de las mejores fuentes de información para evaluar la calidad eléctrica de una fuente.

## Errores comunes

**Subdimensionar la fuente:** el error más frecuente es elegir una fuente con potencia insuficiente para los componentes del sistema. Esto puede causar bloqueos bajo carga, reinicios inesperados o daños permanentes en la fuente y los componentes.

**Confiar en calculadoras online sin verificar:** las calculadoras de potencia son útiles como estimación inicial, pero no sustituyen la consulta de las especificaciones reales de cada componente. El consumo real puede variar significativamente según la carga de trabajo, el overclock y las configuraciones de energía.

**Ignorar la calidad por la potencia:** una fuente genérica de 700W puede no ser tan fiable como una marca reconocida de 550W. La calidad de los componentes internos, las protecciones y la certificación de eficiencia son tan importantes como la potencia nominal.

**No considerar futuras ampliaciones:** si se prevé añadir una segunda GPU, más memoria o almacenamiento adicional, es prudente elegir una fuente con capacidad suficiente para la configuración futura, no solo para la actual.

