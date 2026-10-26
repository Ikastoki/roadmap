# Concepto de almacenamiento

El almacenamiento es el componente de un sistema informático encargado de retener datos de forma permanente o semipermanente, preservándolos más allá del ciclo de encendido y apagado del equipo. A diferencia de la memoria RAM, que pierde su contenido al cortar el suministro eléctrico, los dispositivos de almacenamiento mantienen la información durante años sin energía, constituyendo la capa de persistencia que permite conservar el sistema operativo, las aplicaciones y los datos del usuario.

La evolución del almacenamiento informático es una de las historias más dinámicas de la tecnología. Desde los primeros discos magnéticos IBM RAMAC de 1956 —que pesaban casi una tonelada y almacenaban apenas 5 MB— hasta los modernos SSD NVMe capaces de transferir más de 7 GB/s, el almacenamiento ha experimentado mejoras de varios órdenes de magnitud en capacidad, velocidad y eficiencia. Comprender los principios que subyacen a las diferentes tecnologías de almacenamiento permite tomar decisiones informadas sobre cómo configurar y proteger los datos más valiosos.

## Almacenamiento y persistencia

La persistencia es la propiedad fundamental que distingue al almacenamiento de la memoria volátil. Cuando un usuario guarda un archivo, ese dato debe permanecer accesible durante meses o años, independientemente de si el equipo está encendido o apagado. El almacenamiento proporciona esta persistencia utilizando tecnologías que retienen la información de forma indefinida sin energía.

La distinción entre almacenamiento y memoria va más allá de la volatilidad. El almacenamiento está diseñado para contener grandes volúmenes de datos a un coste por gigabyte mucho menor que la RAM. Un módulo de 16 GB de DDR5 puede costar más de 100 euros, mientras que un SSD de 1 TB cuesta alrededor de 80 euros —una relación de coste de aproximadamente 500:1 por gigabyte. Esta diferencia se debe a que las tecnologías de almacenamiento sacrifican velocidad y latencia a cambio de densidad y economy.

> La relación entre memoria y almacenamiento no es de sustitución sino de complemento. Un sistema eficiente utiliza la memoria como espacio de trabajo de alta velocidad para los datos activos y el almacenamiento como repositorio permanente para todo lo demás.

## Evolución del almacenamiento

La historia del almacenamiento informático comienza con los discos magnéticos de tambor y posteriormente de plato, que dominaron durante décadas. Los discos duros (*Hard Disk Drives*, HDD) utilizan platillos giratorios recubiertos de una capa magnética y cabezales de lectura/escritura que se desplazan sobre la superficie para acceder a los datos. Esta tecnología, aunque madura y económica, está limitada por su naturaleza mecánica: las partes móviles introducen latencia, ruido y un punto de fallo potencial.

La revolución del estado sólido llegó con la memoria NAND Flash, que elimina por completo las partes mecánicas. Los SSD (*Solid State Drives*) almacenan datos en celdas de memoria flash sin moving parts, ofreciendo latencias órdenes de magnitud menores, mayor resistencia a golpes y vibraciones, menor consumo energético y silencio absoluto. La transición de HDD a SSD como unidad de sistema operativo ha sido una de las mejoras de rendimiento más significativas en la historia del PC personal.

Más recientemente, tecnologías como Intel Optane (3D XPoint) y la memoria no volátil MRAM han buscado combinar las ventajas de ambas aproximaciones: la velocidad de los SSD con la persistencia mejorada y la resistencia a la escritura de nuevas tecnologías. Aunque Optane fue discontinuado por Intel, su legado influye en el diseño de las generaciones futuras de almacenamiento.

## Velocidad

La velocidad de un dispositivo de almacenamiento se mide por dos métricas principales: la tasa de transferencia secuencial (cuántos datos se pueden leer o escribir continuamente) y la latencia de acceso (cuánto tiempo tarda el dispositivo en localizar y entregar un dato específico).

Los HDD mecánicos modernos alcanzan velocidades secuenciales de 150-250 MB/s, con latencias del orden de 5-10 milisegundos debidas al tiempo mecánico de posicionamiento de los cabezales. Los SSD SATA alcanzan 500-550 MB/s con latencias de 0.1 ms, mientras que los SSD NVMe sobre PCIe 4.0 superan los 5,000 MB/s con latencias de apenas 0.02 ms.

Estas diferencias tienen un impacto práctico enorme en tareas como el arranque del sistema operativo (de 30-60 segundos con HDD a 10-15 segundos con SSD), la carga de aplicaciones y la apertura de archivos grandes. Para tareas secuenciales grandes, como la transferencia de archivos de vídeo, la diferencia entre HDD y SSD NVMe puede ser de más de 20 veces.

## Capacidad

Las capacidades de almacenamiento modernas varían desde unos pocos gigabytes en dispositivos embebidos hasta varios terabytes en unidades de consumo. Los HDD de escritorio alcanzan los 20-24 TB, los SSD SATA van de 128 GB a 4 TB, y los SSD NVMe de alta gama llegan a 8 TB en formato消费.

La capacidad disponible ha crecido de forma constante, siguiendo una trayectoria similar a la Ley de Moore pero impulsada por diferentes principios físicos. Los HDD aumentan la capacidad incrementando la densidad de platos y la densidad de bits por superficie mediante tecnologías como SMR (*Shingled Magnetic Recording*) y HAMR (*Heat-Assisted Magnetic Recording*). Los SSD la incrementan apilando más capas de celdas NAND (de 2D a 3D con 100+ capas) y utilizando celdas de mayor densidad.

## Velocidad vs capacidad

La elección entre velocidad y capacidad es uno de los trade-offs fundamentales del almacenamiento. Un SSD NVMe de 1 TB ofrece rendimiento excepcional pero un coste por gigabyte mayor que un HDD de 6 TB. La configuración óptima del almacenamiento depende de las necesidades específicas del usuario:

Para sistemas operativos y aplicaciones, donde la velocidad de carga y la respuesta son críticas, un SSD es prácticamente indispensable. Para archivos multimedia, copias de seguridad y datos de archivo que no requieren acceso rápido, un HDD sigue siendo una opción económica y fiable.

La configuración más común en la práctica combina ambos: un SSD como unidad de sistema (donde residen el SO y las aplicaciones más usadas) y un HDD como almacenamiento secundario para archivos grandes y datos de archivo. Esta aproximación maximiza el rendimiento percibido mientras se mantiene una capacidad generosa a un coste razonable.

