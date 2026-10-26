# Buses externos

Los buses externos son los estándares de comunicación que permiten conectar periféricos y dispositivos externos a una computadora. A diferencia de los buses internos, diseñados para máxima velocidad dentro del chasis, los buses externos equilibran rendimiento con usabilidad: deben ser fáciles de conectar/desconectar, tolerantes a una variedad de condiciones de uso y compatibles con una amplia gama de dispositivos que van desde ratones simples hasta estaciones de acoplamiento de alta velocidad.

La evolución de los buses externos refleja la creciente demanda de ancho de banda por parte de los periféricos modernos. Lo que comenzó con el puerto serie y paralelo de baja velocidad ha evolucionado hacia interfaces como USB 4 y Thunderbolt, capaces de transportar vídeo, datos y energía a través de un único cable con anchos de banda que superan los 40 GB/s.

## Interfaz USB

USB (*Universal Serial Bus*) es el estándar de conexión externa más extendido del mundo. Introducido en 1996, unificó la proliferación de puertos serie, paralelo y PS/2 que existían en los PC de la época, proporcionando un conector único y estandarizado que simplificó enormemente la conexión de periféricos.

### Tipos de conectores (A, B, C)

**USB Tipo A:** el conector rectangular clásico que se encuentra en la mayoría de ordenadores y cargadores. Es el conector más reconocible y universal, pero solo permite la conexión en una orientación (no es reversible). Se utiliza como puerto de anfitrión (*host*) en prácticamente todos los dispositivos de computación.

**USB Tipo B:** el conector cuadrado utilizado típicamente en impresoras, escáneres y algunos dispositivos de almacenamiento externo. Existen variantes para USB 2.0 (cuadrado) y USB 3.0 (más ancho y azul). Es menos común que el Tipo A y está siendo progresivamente reemplazado por el Tipo C.

**USB Tipo C:** el conector reversible ovalado que se ha convertido en el estándar moderno. Su diseño simétrico permite la inserción en cualquier orientación, eliminando la frustración del "conector que nunca encaja bien a la primera". USB-C es capaz de transportar datos, vídeo (DisplayPort Alt Mode), energía (USB Power Delivery) y señal de audio a través de un único conector, lo que lo convierte en la interfaz más versátil disponible.

### Estándares (USB 2.0, 3.x, 4)

**USB 2.0 (2000):** ofrece un ancho de banda de 480 Mbps (60 MB/s teórico). A pesar de su antigüedad, sigue siendo ampliamente utilizado para dispositivos de bajo ancho de banda como ratones, teclados y dispositivos de audio. La velocidad real típica ronda los 25-30 MB/s debido a la sobrecarga del protocolo.

**USB 3.0 / USB 3.1 Gen 1 (2008):** incrementó el ancho de banda a 5 Gbps (500 MB/s teórico), una mejora de más de 10 veces respecto a USB 2.0. Los cables y puertos USB 3.0 se identifican típicamente por su color azul. La velocidad real alcanza los 400-450 MB/s.

**USB 3.1 Gen 2 / USB 3.2 Gen 2 (2013):** duplicó la velocidad a 10 Gbps (1,250 MB/s teórico), acercándose a las velocidades de los SSD NVMe externos. La velocidad real ronda los 900-1,000 MB/s.

**USB 3.2 Gen 2x2 (2017):** utiliza dos carriles simultáneos para alcanzar 20 Gbps (2,500 MB/s teórico). Requiere conector USB-C y es la primera versión que utiliza múltiples carriles en USB.

**USB4 (2019):** alcanza 40 Gbps (5,000 MB/s teórico) y adopta el protocolo de túnel de Thunderbolt, permitiendo multiplexar USB, DisplayPort y PCIe sobre el mismo cable. USB4 v2 (2022) duplica la velocidad a 80 Gbps.

### Transferencia de datos

La velocidad real de transferencia de USB depende no solo del estándar sino también de la calidad del cable, la implementación del controlador y la carga de trabajo. Los cables de mala calidad o excesivamente largos pueden no soportar la velocidad completa del estándar, resultando en transferencias más lentas. Para USB 3.0 y posteriores, se recomienda utilizar cables certificados que cumplan con las especificaciones del estándar.

## Thunderbolt

Thunderbolt es una interfaz de alta velocidad desarrollada originalmente por Intel y Apple que combina PCIe, DisplayPort y energía en un único conector. Desde su tercera generación, Thunderbolt utiliza el conector USB-C, ofreciendo compatibilidad física con los puertos USB-C mientras proporciona funcionalidades que USB-C estándar no incluye.

**Thunderbolt 3 (2015):** ofrece 40 Gbps de ancho de banda, suficiente para conectar múltiples monitores 4K, SSD externos de alta velocidad y estaciones de acoplamiento con múltiples periféricos a través de un único cable. Es retrocompatible con USB-C y DisplayPort.

**Thunderbolt 4 (2020):** mantiene los 40 Gbps pero eleva los requisitos mínimos de rendimiento: soporte obligatorio para dos monitores 4K o uno de 8K, y un ancho de banda PCIe mínimo de 32 Gbps para dispositivos de almacenamiento (frente a los 16 Gbps mínimos de Thunderbolt 3).

**Thunderbolt 5 (2024):** alcanza 80 Gbps de ancho de banda (y 120 Gbps en modo unidireccional para vídeo), suficiente para múltiples monitores 8K o SSD externos que igualan la velocidad de los internos. Thunderbolt 5 utiliza la señalización PAM-4 para duplicar la tasa de datos sin aumentar la frecuencia del cable.

> Thunderbolt es particularmente relevante para portátiles, donde un único puerto Thunderbolt puede conectarse a una estación de acoplamiento que proporciona acceso a monitores, red, almacenamiento y periféricos de escritorio a través de un solo cable. Esta capacidad de "conectar todo con un cable" ha convertido a Thunderbolt en un estándar fundamental para la productividad profesional.

## SATA

SATA (*Serial Advanced Technology Attachment*) es la interfaz de conexión estándar para dispositivos de almacenamiento de consumo. Introducida en 2003 como sucesora del paralelo ATA (IDE), SATA es un bus serial point-to-point que conecta cada dispositivo directamente al controlador, eliminando las complicaciones de la configuración maestro/esclavo del IDE.

**SATA I (1.5 Gbps):** la primera generación, con un ancho de banda de 150 MB/s. Aunque obsoleta para SSD, sigue siendo adecuada para HDD, cuya velocidad secuencial no supera los 250 MB/s.

**SATA II (3 Gbps):** duplicó la velocidad a 300 MB/s, suficiente para los SSD SATA que alcanzan 550 MB/s en lectura (la diferencia se debe a que SATA II es la velocidad de línea bruta, mientras que 550 MB/s es la transferencia efectiva con codificación 8b/10b).

**SATA III (6 Gbps):** la versión actual, con 600 MB/s de ancho de banda de línea. Es la interfaz estándar para SSD SATA y la mayoría de los HDD modernos. Aunque SATA III sigue siendo ampliamente utilizado, está siendo progresivamente desplazado por NVMe en los nuevos sistemas.

La limitación fundamental de SATA es su diseño point-to-point: cada dispositivo SATA necesita su propio puerto en la placa base, y el ancho de banda no es compartible entre dispositivos. Con 6 puertos SATA III en una placa base típica, el ancho de banda total disponible es de 3.6 GB/s, insuficiente para múltiples SSD NVMe que individualmente pueden superar los 7 GB/s.

> SATA sigue siendo relevante para HDD de gran capacidad, SSD SATA en sistemas antiguos y unidades de disco óptico. Sin embargo, para nuevos sistemas, NVMe sobre M.2 ofrece un rendimiento significativamente superior al mismo precio, lo que está acelerando la transición de SATA a NVMe como estándar de almacenamiento.

