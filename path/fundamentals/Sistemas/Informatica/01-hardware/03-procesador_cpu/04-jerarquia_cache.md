# La jerarquía de caché

La memoria caché es el componente que más impacto tiene en el rendimiento percibido de un procesador. Situada entre la CPU y la memoria principal, la caché almacena copias de los datos e instrucciones más utilizados, permitiendo al procesador acceder a ellos en nanosegundos en lugar de los cientos de nanosegundos que requiere la memoria principal. Sin una caché eficiente, incluso el procesador más rápido pasaría la mayoría de su tiempo esperando datos.

La jerarquía de caché se estructura en múltiples niveles, cada uno con diferentes características de tamaño, velocidad y proximidad al núcleo del procesador. Comprender esta jerarquía es fundamental para entender cómo se comportan las aplicaciones a nivel de rendimiento.

## Caché L1, L2, L3

Los caches se organizan en niveles jerárquicos, donde el nivel 1 (L1) es el más rápido y pequeño, y el nivel 3 (L3) es el más lento y grande:

### Caché L1

La caché L1 es la más cercana al núcleo del procesador y la más rápida. Se encuentra integrada físicamente dentro del núcleo y se accede en 1-4 ciclos de reloj, lo que equivale a aproximadamente 1-2 nanosegundos en un procesador a 3 GHz.

Cada núcleo típicamente tiene dos caches L1 separadas:

- **L1 I-Caché (instrucciones):** almacena las instrucciones del programa. Solo es legible.
- **L1 D-Caché (datos):** almacena los datos con los que opera la CPU. Es legible y escribible.

El tamaño típico de la caché L1 varía entre 32 KB y 128 KB por núcleo. Los fabricantes mantienen la caché L1 relativamente pequeña porque su proximidad al núcleo implica que cada bit de capacidad adicional ocupa espacio valioso en el chip y aumenta el consumo energético.

### Caché L2

La caché L2 es el nivel intermedio de la jerarquía. Es significativamente más grande que la L1 —típicamente entre 256 KB y 2 MB por núcleo— pero también más lenta, con latencias de 10-20 ciclos de reloj.

En los procesadores modernos, la caché L2 suele ser exclusiva por núcleo, lo que significa que cada núcleo tiene su propia caché L2 dedicada. Esto evita las interferencias entre núcleos pero limita la capacidad total de caché disponible.

Cuando un dato solicitado por la CPU no se encuentra en la L1 (*cache miss*), el procesador busca en la L2. Si lo encuentra (*L2 hit*), el dato se trae a la L1 y se utiliza. Si tampoco está en la L2, se busca en la L3 o, en último caso, en la memoria principal.

### Caché L3

La caché L3 es el nivel más grande y el más cercano a la memoria principal. Su capacidad varía entre 4 MB y cientos de MB en procesadores de gama alta, con latencias de 30-50 ciclos de reloj.

A diferencia de L1 y L2, la caché L3 es compartida entre todos los núcleos del procesador. Esto permite que los datos sean accesibles por cualquier núcleo, facilitando la comunicación entre núcleos y reduciendo la necesidad de acceder a la memoria principal para transferir datos entre núcleos.

> La caché L3 ha crecido enormemente en los últimos años. Los procesadores AMD Ryzen de 3ª generación incorporan hasta 64 MB de caché L3, y los Intel Core de 13ª generación alcanzan los 36 MB. Los AMD Ryzen con 3D V-Cache llegan hasta 128 MB de L3, demostrando el impacto directo de la caché en el rendimiento de ciertas cargas de trabajo.

## Latencia y velocidad

La latencia de acceso a cada nivel de la jerarquía varía enormemente:

| Nivel | Latencia (ciclos) | Latencia (ns aprox.) | Tamaño típico |
|---|---|---|---|
| Registros | 0-1 | 0.3 | Cientos de bytes |
| L1 | 1-4 | 1-2 | 32-128 KB |
| L2 | 10-20 | 3-7 | 256 KB-2 MB |
| L3 | 30-50 | 10-20 | 4-128 MB |
| RAM | 50-100+ | 20-100+ | 8-128 GB |

Estas diferencias de latencia explican por qué un cache hit es tan valioso: acceder a L1 es aproximadamente 100 veces más rápido que acceder a la memoria principal. En una aplicación que ejecuta miles de millones de operaciones por segundo, esta diferencia se traduce en un impacto enorme en el rendimiento.

## Importancia para el rendimiento

La eficiencia de la caché determina el rendimiento real de un procesador. Un procesador con alto IPC y baja latencia de caché puede superar a uno con mayor frecuencia de reloj pero peor gestión de caché.

El concepto de *Working Set* describe el conjunto de datos que una aplicación necesita acceder frecuentemente. Si el Working Set cabe en la caché L3, la aplicación tendrá un rendimiento excelente. Si excede la capacidad de la L3, se producirán misses frecuentes en la memoria principal, degradando significativamente el rendimiento.

Las aplicaciones se benefician de diferentes formas según su patrón de acceso a memoria:

- **Aplicaciones de oficina:** Working Set pequeño, alta localidad temporal. Se benefician de caches L1 y L2 eficientes.
- **Bases de datos:** Working Set grande, acceso pseudoaleatorio. Se benefician de grandes caches L3.
- **Juegos:** Working Set moderado, alta localidad espacial. Se benefician de caches de alto ancho de banda.
- **IA y machine learning:** Working Set enorme, acceso secuencial. Se benefician de ancho de banda de memoria más que de latencia de caché.

> Los benchmarks de rendimiento que ignoran el comportamiento de la caché pueden ser engañosos. Un procesador puede tener excelente IPC en pruebas sintéticas pero rendimiento deficiente en aplicaciones reales si su caché no es suficientemente grande o rápida para el Working Set de la aplicación.

