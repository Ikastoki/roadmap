# RAM

La memoria de acceso aleatorio (*Random Access Memory*) es el componente que almacena temporalmente los datos e instrucciones que el procesador necesita para ejecutar programas. A diferencia del almacenamiento permanente, la RAM permite tanto lectura como escritura a prácticamente cualquier dirección de memoria en el mismo tiempo, sin importar la posición física donde se encuentren los datos. Esta propiedad es la que la hace ideal como espacio de trabajo del sistema.

La RAM ha evolucionado enormemente desde sus orígenes en las décadas de 1960 y 1970. Los primeros sistemas utilizaban núcleos de ferrita, posteriormente fueron reemplazados por circuitos integrados de memoria, y las generaciones de DRAM (*Dynamic RAM*) han ido aumentando la velocidad y reduciendo el consumo energético. La tecnología actual se basa en memorias DRAM, que almacenan cada bit de información en un condensador que debe ser periódicamente recargado para conservar el dato.

## Funcionamiento

La memoria RAM se compone de miles de millones de celdas, cada una formada por un condensador y un transistor. El condensador almacena el bit de información como carga eléctrica (cargado = 1, descargado = 0), y el transistor actúa como un interruptor que permite leer o escribir el estado del condensador.

Debido a que el condensador pierde carga de forma natural con el tiempo, las memorias DRAM requieren refresco periódico: cada ciertos milisegundos, un circuito de refresco recarga todas las celdas para evitar la pérdida de datos. Este mecanismo de refresco es lo que distingue a la DRAM de la SRAM (*Static RAM*), que mantiene el dato mientras reciba energía sin necesidad de refresco, pero que es más costosa y ocupa más espacio物理.

> La naturaleza dinámica de la DRAM (de ahí la "D" en DRAM) implica que los datos no están verdaderamente "estáticos" en memoria: dependen de un ciclo continuo de recarga que ocurre invisiblemente para el software.

## Tipos de RAM

Existen múltiples tecnologías de RAM, cada una diseñada para un conjunto diferente de requisitos de rendimiento, consumo y coste:

**SRAM (*Static RAM*):** utiliza-flip flops (circuitos de retroalimentación) en lugar de condensadores, lo que elimina la necesidad de refresco. Es mucho más rápida que la DRAM (latencias de 1-2 ns vs. 50-100 ns), pero también considerablemente más cara y menos densa. Se utiliza principalmente para las cachés del procesador (L1, L2 y部分 L3).

**DRAM (*Dynamic RAM*):** la tecnología base de la memoria principal. Ofrece un buen equilibrio entre densidad, velocidad y coste. La DRAM moderna se comercializa en variantes sincronizadas (SDRAM) que operan sincronizadamente con el reloj del sistema.

**DDR SDRAM (*Double Data Rate*):** evolución de la SDRAM que transfiere datos en ambos flancos del pulso de reloj (subida y bajada), duplicando el ancho de banda efectivo sin aumentar la frecuencia del reloj. Es la tecnología dominante en la memoria de escritorio y portátil actual.

## DDR: generaciones

Las generaciones de DDR representan los hitos más importantes en la evolución de la memoria de consumo:

**DDR (generación 1):** lanzada en 2000, operaba a 100-200 MHz (DDR-200 a DDR-400) con voltajes de 2.5V. Utilizaba 184 pines en formato DIMM para escritorio y 200 pines en formato SO-DIMM para portátiles.

**DDR2:** introducida en 2003, duplicó la frecuencia efectiva respecto a DDR (400-1066 MHz) reduciendo el voltaje a 1.8V. Aumentó los pines a 240 en DIMM, mejoró la eficiencia energética y añadió buffers en los módulos para reducir la carga del controlador de memoria.

**DDR3:** llegada en 2007, ofreció frecuencias de 800-2133 MHz con voltaje de 1.5V (y 1.35V en variantes de bajo consumo). Los módulos DIMM mantuvieron 240 pines pero con un slot incompatible con DDR2, evitando la inserción incorrecta.

**DDR4:** desde 2014, opera entre 1600-3200 MHz (y más allá con overclock) con voltaje de 1.2V. Los módulos DIMM pasaron a 288 pines con un contacto cóncavo para diferenciarlos visualmente de generaciones anteriores. DDR4 introdujo mejoras significativas en eficiencia energética y densidad, con módulos de hasta 64 GB.

**DDR5:** lanzada en 2021, comienza en 4800 MHz y alcanza más de 8000 MHz. El voltaje se reduce a 1.1V, y los módulos DIMM mantienen 288 pines pero con un diseño diferente. DDR5 incorpora un controlador de memoria en el propio módulo (*on-die ECC*) y soporte nativo para canales de 32 bits (frente a los 64 bits de DDR4), permitiendo mejoras en rendimiento y fiabilidad.

## Diferencias en pines, voltaje y velocidad

| Generación | Pines DIMM | Voltaje | Frecuencia típica |
|------------|-----------|---------|-------------------|
| DDR | 184 | 2.5V | 200-400 MHz |
| DDR2 | 240 | 1.8V | 400-1066 MHz |
| DDR3 | 240 | 1.5V | 800-2133 MHz |
| DDR4 | 288 | 1.2V | 1600-3200 MHz |
| DDR5 | 288 | 1.1V | 4800-8000+ MHz |

Las generaciones de DDR no son compatibles entre sí: cada una utiliza un slot con una muesca de alineación en una posición diferente, impidiendo físicamente la inserción de un módulo incompatible. Esta protección mecánica evita daños por voltajes o frecuencias incompatibles.

El voltaje decreciente entre generaciones refleja la tendencia hacia la eficiencia energética. Un módulo DDR4-3200 consume aproximadamente la mitad de energía por bit transferido que un módulo DDR3-1600, mientras ofrece el doble de ancho de banda.

> La tendencia de cada generación a reducir el voltaje mientras aumenta la frecuencia es un equilibrio entre rendimiento y thermal design. Los voltajes más bajos generan menos calor, lo que permite densidades mayores y frecuencias más altas sin sobrecalentamiento.

