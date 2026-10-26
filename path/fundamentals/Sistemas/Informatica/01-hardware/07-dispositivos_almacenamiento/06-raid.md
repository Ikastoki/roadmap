# RAID: redundancia y rendimiento

RAID (*Redundant Array of Independent Disks*) es una tecnología que combina múltiples unidades de almacenamiento en un solo conjunto lógico para mejorar el rendimiento, la fiabilidad o ambas cosas simultáneamente. Desarrollada inicialmente en 1987 por David Patterson, Garth Gibson y Randy Katz en la Universidad de California, Berkeley, RAID transformó la idea de que los discos pequeños y rápidos eran superiores a los grandes y lentos, demostrando que un arreglo de discos económicos podía superar en rendimiento y fiabilidad a los discos individuales más costosos.

La implementación de RAID permite que el sistema operativo vea múltiples discos físicos como una sola unidad lógica, mientras el controlador RAID gestiona la distribución de datos entre los discos individuales. Cada nivel de RAID ofrece un equilibrio diferente entre rendimiento, tolerancia a fallos y eficiencia de almacenamiento, y la elección del nivel adecuado depende de los requisitos específicos de la aplicación.

## RAID 0, 1, 5, 10

**RAID 0 (*Striping*):** distribuye los datos en bloques alternos entre los discos del conjunto, doblando (o más) la velocidad de transferencia al permitir lecturas y escrituras simultáneas en múltiples discos. Con dos discos, cada bloque de datos se divide en dos mitades que se escriben simultáneamente, alcanzando una velocidad teórica del doble de un solo disco.

```
Disco 1: [Bloque 1] [Bloque 3] [Bloque 5]
Disco 2: [Bloque 2] [Bloque 4] [Bloque 6]
```

El RAID 0 ofrece la máxima eficiencia de almacenamiento (100% del espacio usable) y el mejor rendimiento secuencial, pero no tiene redundancia: si un solo disco falla, se pierden todos los datos del conjunto. Es adecuado para almacenamiento temporal o caché donde la pérdida de datos es aceptable y la velocidad es prioritaria.

**RAID 1 (*Mirroring*):** duplica cada escribir en dos discos simultáneamente, creando un espejo idéntico de los datos. Si un disco falla, el otro contiene una copia completa que permite la operación continua sin pérdida de datos.

```
Disco 1: [Bloque 1] [Bloque 2] [Bloque 3]
Disco 2: [Bloque 1] [Bloque 2] [Bloque 3]
```

El RAID 1 proporciona la mejor tolerancia a fallos (puede sobrevivir la pérdida de cualquier disco), pero reduce la capacidad efectiva a la mitad (solo se usa la capacidad de un disco). La velocidad de lectura puede mejorar al poder leer de ambos discos simultáneamente, pero la velocidad de escritura no aumenta ya que cada dato debe escribirse en ambos discos.

**RAID 5 (*Striping with Parity*):** combina striping con información de paridad distribuida entre los discos. La paridad es un cálculo matemático que permite reconstruir los datos de un disco faltante a partir de los demás. Con al menos tres discos, RAID 5 distribuye los datos y la paridad de forma rotativa, ofreciendo un buen equilibrio entre rendimiento, capacidad y tolerancia a fallos.

```
Disco 1: [Bloque 1] [Bloque 2] [Paridad 3]
Disco 2: [Bloque 2] [Paridad 1] [Bloque 3]
Disco 3: [Paridad 2] [Bloque 3] [Bloque 1]
```

El RAID 5 permite la pérdida de un disco sin pérdida de datos, con una eficiencia de almacenamiento de (N-1)/N (con 4 discos, el 75% del espacio total es usable). La velocidad de lectura es excelente (similar a RAID 0 con N-1 discos), pero la escritura tiene una penalización por los cálculos de paridad. Es el nivel más común para servidores de archivos y almacenamiento general.

**RAID 10 (*Stripe of Mirrors*):** combina RAID 1 y RAID 0, creando pares de discos espejados que luego se distribuyen en striping. Es la combinación de rendimiento y fiabilidad más robusta disponible, pero requiere un mínimo de 4 discos y tiene una eficiencia de capacidad del 50%.

```
Par 1 (Espejo):  [Disco 1A] [Disco 1B]
Par 2 (Espejo):  [Disco 2A] [Disco 2B]
Striping entre pares: Disco 1A/1B ↔ Disco 2A/2B
```

El RAID 10 ofrece excelente rendimiento tanto de lectura como de escritura, y puede sobrevivir al fallo de al menos un disco por par. Es la opción preferida para bases de datos transaccionales, virtualización y aplicaciones donde tanto el rendimiento como la disponibilidad son críticos.

## Implementación por hardware

El RAID por hardware utiliza un controlador RAID dedicado, una tarjeta de expansión PCIe o un chip integrado en la placa base que gestiona todas las operaciones de RAID sin carga del procesador principal. Los controladores RAID de hardware suelen ofrecer mejor rendimiento, caché de escritura con protección por batería (BBU o supercondensador) y soporte para niveles avanzados de RAID.

Los controladores RAID de gama alta (LSI, Adaptec, HighPoint) incluyen procesadores dedicados y caché de escritura de hasta 4 GB que buffer las escrituras antes de commitarlas al disco, mejorando significativamente el rendimiento de escritura aleatoria. La caché protegida por batería garantiza que los datos en caché no se pierdan en caso de corte de energía.

La implementación por hardware es prácticamente obligatoria en servidores de producción, donde la fiabilidad, el rendimiento y la gestión de múltiples conjuntos RAID son requisitos críticos.

## Implementación por software

El RAID por software se implementa a nivel del sistema operativo, utilizando utilidades como `mdadm` en Linux, Storage Spaces en Windows o ZFS en Solaris/FreeBSD/Linux. El procesador del sistema se encarga de todos los cálculos de striping, paridad y gestión de errores, lo que añade una carga adicional al CPU.

Las ventajas del RAID por software incluyen la ausencia de coste adicional por hardware, la portabilidad (los conjuntos RAID pueden trasladarse entre sistemas sin depender de un controlador específico) y la flexibilidad en la gestión. En sistemas modernos con procesadores potentes, la diferencia de rendimiento entre RAID por hardware y por software es menor que en el pasado, especialmente para niveles simples como RAID 1 y RAID 5.

> ZFS, aunque no es estrictamente RAID, implementa funciones equivalentes (RAID-Z, RAID-Z2, RAID-Z3) con ventajas significativas: integridad de datos en tiempo real, compresión transparente, snapshots y clonación. ZFS RAID-Z2 y RAID-Z3 permiten la pérdida de dos y tres discos respectivamente, superando la tolerancia de RAID 5 y RAID 6 tradicionales.

