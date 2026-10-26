# Buses internos

Los buses internos son los sistemas de comunicación que conectan los componentes principales dentro de una computadora: el procesador, la memoria RAM, los dispositivos de almacenamiento y las tarjetas de expansión. A diferencia de los buses externos, que conectan periféricos fuera del chasis, los buses internos están diseñados para máxima velocidad y mínima latencia, ya que transportan los flujos de datos más críticos del sistema.

La arquitectura de los buses internos ha evolucionado enormemente desde los primeros sistemas con buses compartidos, donde todos los componentes competían por el mismo medio, hasta las arquitecturas actuales con enlaces point-to-point dedicados que conectan directamente cada componente con el controlador correspondiente. Esta evolución ha sido impulsada por la necesidad de alimentar cada vez más ancho de banda a los componentes más demandantes, particularmente las GPU y los dispositivos de almacenamiento NVMe.

## PCI Express

PCI Express (*Peripheral Component Interconnect Express*) es el estándar de interconexión de alto rendimiento que domina actualmente la industria del hardware. Sustituyó a los buses PCI y PCI-X convencionales mediante una arquitectura serial de alta velocidad que ofrece mayor ancho de banda, menor latencia y escalabilidad mediante la adición de carriles.

A diferencia del PCI original, que utilizaba un bus paralelo compartido, PCI Express establece enlaces point-to-point entre el dispositivo y el host. Cada enlace se compone de uno o más carriles (*lanes*), donde cada carril es un enlace serial bidireccional independiente. Los dispositivos se conectan mediante enlaces de x1, x4, x8 o x16 carriles, y los slots de la placa base están físicamente dimensionados para acomodar los diferentes anchos.

| Generación | Velocidad por carril | x16 (teórico) | x4 (teórico) |
|------------|---------------------|---------------|--------------|
| PCIe 1.0 | 2.5 GT/s | 8 GB/s | 2 GB/s |
| PCIe 2.0 | 5 GT/s | 16 GB/s | 4 GB/s |
| PCIe 3.0 | 8 GT/s | 31.5 GB/s | ~4 GB/s |
| PCIe 4.0 | 16 GT/s | 63 GB/s | ~8 GB/s |
| PCIe 5.0 | 32 GT/s | 126 GB/s | ~16 GB/s |
| PCIe 6.0 | 64 GT/s | 252 GB/s | ~32 GB/s |

PCIe es retrocompatible: un dispositivo PCIe 3.0 funcionará en un slot PCIe 4.0 a la velocidad de su propia generación. Esto permite la actualización incremental de componentes sin necesidad de reemplazar toda la placa base.

> Los slots físicos PCIe de diferentes tamaños (x1, x4, x8, x16) son mecánicamente compatibles hacia abajo: un dispositivo x1 puede insertarse en un slot x4, x8 o x16, funcionando a la velocidad del enlace más corto disponible. Esta flexibilidad permite instalar dispositivos en slots que no coinciden exactamente con su factor de forma.

## Bus de memoria

El bus de memoria es la conexión entre el controlador de memoria integrado en el procesador y los módulos de RAM instalados en la placa base. Su importancia es capital: toda la información que la CPU procesa debe pasar por este bus en algún momento, y su ancho de banda determina cuántos datos pueden fluir entre la memoria y el procesador.

En los sistemas modernos, el controlador de memoria está integrado directamente en el procesador (una tendencia que comenzó con la primera generación de Intel Core y AMD Ryzen), eliminando la necesidad de un chip externo en la placa base y reduciendo significativamente la latencia de acceso a la memoria.

El ancho del bus de memoria es proporcional al número de canales. Un sistema dual channel con DDR4-3200 tiene un bus de 128 bits (64 bits por canal) que opera a 1,600 MHz, proporcionando 51.2 GB/s de ancho de banda teórico. El bus de memoria es típicamente el cuello de botella más significativo en un sistema de escritorio, ya que su ancho de banda es varias veces menor que el de la caché L3 del procesador.

La arquitectura del bus de memoria también define el interleave: en un sistema dual channel, las direcciones pares van a un canal y las impares al otro, distribuyendo la carga y duplicando el ancho de banda efectivo. Para que el interleave funcione correctamente, los módulos deben instalarse en los slots correctos según la configuración de la placa base.

## Bus del sistema

El bus del sistema, también conocido como *front-side bus* (FSB) en la terminología clásica, era originalmente la conexión principal entre el procesador y el chipset de la placa base. En la arquitectura moderna, con el controlador de memoria y el controlador PCIe integrados en el procesador, el concepto de bus del sistema se ha fragmentado en múltiples enlaces dedicados.

En los sistemas actuales, las conexiones más relevantes dentro del sistema son:

**Enlaces CPU-chipset (*DMI, Direct Media Interface*):** en las plataformas Intel, el procesador se comunica con el chipset (PCH, *Platform Controller Hub*) mediante el enlace DMI. En la versión actual (DMI 4.0), proporciona un ancho de banda de 8 GB/s (equivalente a PCIe 3.0 x8). Este enlace transporta la comunicación hacia los dispositivos conectados al chipset: puertos USB, puertos SATA, slots PCIe del chipset y audio integrado.

**Infinity Fabric (AMD):** en las plataformas AMD Ryzen y EPYC, el Infinity Fabric es el sistema de interconexión que conecta los núcleos del procesador entre sí, con la caché L3 y con el controlador de memoria y PCIe. El rendimiento del Infinity Fabric está directamente ligado a la frecuencia de la memoria, lo que explica por qué la velocidad de la RAM tiene un impacto significativo en el rendimiento de los procesadores AMD.

**Ring Bus y Mesh:** dentro del propio procesador, los núcleos se comunican entre sí y con la caché compartida mediante buses internos como el ring bus (anel circular) o la malla (*mesh*). Intel ha utilizado ambas arquitecturas en diferentes generaciones de procesadores, con el mesh ofreciendo mejor escalabilidad en diseños con muchos núcleos.

> La tendencia actual es integrar cada vez más funcionalidades dentro del propio procesador, reduciendo la dependencia de buses externos. Los controladores de memoria, PCIe e incluso USB están migrando al die del procesador, acortando las distancias físicas de comunicación y reduciendo la latencia.

