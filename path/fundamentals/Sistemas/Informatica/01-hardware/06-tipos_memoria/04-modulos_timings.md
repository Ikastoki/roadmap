# Módulos y timings

Los módulos de memoria RAM no son componentes genéricos: cada uno está caracterizado por una serie de parámetros de temporización que determinan con qué rapidez puede accederse a los datos almacenados. Comprender estos parámetros —particularmente los *timings* de latencia CAS y los perfiles XMP/EXPO— permite tomar decisiones informadas sobre la selección de memoria y su configuración óptima.

El rendimiento real de un módulo de memoria no se define únicamente por su frecuencia nominal: dos módulos operando a la misma frecuencia pueden comportarse de forma significativamente diferente dependiendo de la configuración de sus timings. Este artículo analiza en detalle los parámetros que gobiernan el acceso a la memoria y cómo afectan al rendimiento práctico.

## Los valores CL (CAS Latency)

La latencia CAS (*Column Address Strobe Latency*), comúnmente abreviada como CL o *CAS Latency*, es el número de ciclos de reloj que la memoria necesita para localizar y entregar los datos después de recibir una solicitud de lectura. Es el timing más conocido y uno de los más importantes para determinar la latencia absoluta de la memoria.

Un módulo DDR4-3200 CL16 tiene una latencia CAS de 16 ciclos de reloj. Para convertir esta cifra en tiempo absoluto:

```
Latencia (ns) = (CL / Frecuencia en MHz) x 1000

CL16 a 3200 MHz = (16 / 3200) x 1000 = 5.0 ns
```

Una memoria DDR4-3200 CL14 tendría la misma frecuencia pero menor latencia:

```
CL14 a 3200 MHz = (14 / 3200) x 1000 = 4.375 ns
```

La diferencia de 0.625 ns puede parecer insignificante, pero se acumula a lo largo de millones de accesos por segundo, impactando notablemente en el rendimiento de aplicaciones sensibles a la latencia como videojuegos, bases de datos y compilación de código.

> Un error frecuente es comparar directamente el valor CL entre memorias de distinta frecuencia. Un módulo DDR4-3600 CL18 no es necesariamente más lento que uno DDR4-3200 CL16, ya que la menor latencia absoluta del CL más bajo puede ser compensada por la mayor frecuencia del otro. La comparación siempre debe hacerse en términos de nanosegundos, no de ciclos.

## Otros timings

Además del CL, los módulos de memoria RAM tienen múltiples timings adicionales que gobiernan diferentes aspectos del ciclo de acceso. Estos timings se expresan típicamente como una secuencia de cuatro números, por ejemplo 16-18-18-36:

**tRCD (*RAS to CAS Delay*):** el tiempo de espera entre la activación de una fila (*Row Address Strobe*) y la solicitud de una columna (*Column Address Strobe*) dentro de esa fila. Representa el tiempo necesario para preparar una fila para el acceso a sus datos.

**tRP (*Row Precharge Time*):** el tiempo necesario para desactivar una fila y preparar la siguiente operación. Determina cuánto debe esperar el controlador antes de poder acceder a una fila diferente en el mismo banco.

**tRAS (*Row Active Time*):** el tiempo mínimo que una fila debe permanecer activa antes de poder ser desactivada. Es la suma de tRCD y el tiempo mínimo de acceso, garantizando que los datos se lean completamente antes de cerrar la fila.

**tRFC (*Refresh Cycle Time*):** el tiempo requerido para el ciclo de refresco de memoria. Este parámetro es especialmente relevante en módulos de alta densidad, donde el refresco de todas las celdas toma más tiempo.

Los timings más bajos son mejores, pero existe un trade-off fundamental entre latencia y estabilidad. Los fabricantes de módulos seleccionan timings que garantizan un funcionamiento estable a la frecuencia anunciada. Reducir los timings manualmente (*underclocking* de timings) puede provocar errores si los valores elegidos están fuera de las especificaciones del chip de memoria.

## Perfiles XMP/EXPO

Los perfiles XMP (*Extreme Memory Profile*, Intel) y EXPO (*Extended Profiles for Overclocking*, AMD) son configuraciones predefinidas en los módulos de memoria que permiten al usuario activar frecuencias y timings superiores a los estándar con un solo ajuste en la BIOS/UEFI.

Sin activar el perfil XMP/EXPO, los módulos de memoria operan a la frecuencia JEDEC estándar (por ejemplo, DDR4-2133 para DDR4 o DDR5-4800 para DDR5), que es conservadora y garantiza compatibilidad universal. Al activar el perfil, la BIOS aplica automáticamente la frecuencia, timings y voltaje especificados por el fabricante para ese módulo concreto:

```
Frecuencia: 3200 MHz (frente a 2133 JEDEC)
Timings: 16-18-18-36
Voltaje: 1.35V (frente a 1.2V JEDEC)
```

XMP es una tecnología propietaria de Intel, mientras que EXPO es la alternativa abierta de AMD. Los módulos modernos suelen incluir perfiles para ambas plataformas, y la mayoría de placas base actuales soportan ambos estándares.

> La activación de XMP/EXPO se considera técnicamente *overclocking*, ya que opera los módulos a frecuencias y voltajes superiores a los estándar JEDEC. Sin embargo, los perfiles están validados por el fabricante y son seguros siempre que la placa base y el procesador soporten la frecuencia objetivo.

