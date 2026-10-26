# Unidades de disco duro (HDD)

El disco duro (*Hard Disk Drive*) es el dispositivo de almacenamiento que durante más de medio siglo ha sido la columna vertebral de la persistencia de datos en la informática. Aunque los SSD han ganado terreno rápidamente como unidad de sistema, los HDD siguen siendo omnipresentes en aplicaciones donde la capacidad y el coste por gigabyte son prioritarios: servidores de archivo, sistemas de copia de seguridad, centros de datos de almacenamiento masivo y cualquier escenario donde se necesiten terabytes de espacio a un precio accesible.

La tecnología HDD es notable por su sofisticación mecánica: componentes que se mueven a miles de revoluciones por minuto con tolerancias de nanómetros, logrando una densidad de almacenamiento que rivaliza con la de los semiconductores. Este artículo analiza los principios físicos que gobiernan el funcionamiento de los HDD y las métricas que determinan su rendimiento.

## Funcionamiento mecánico (platos, cabezales)

Un disco duro está compuesto por uno o más discos o *platillos* (platters) montados en un eje central (*spindle*) que gira a alta velocidad. Cada platillo está recubierto en ambas caras con una capa ferromagnética donde se graban los datos como patrones de magnetización. Los *cabezales* de lectura/escritura se suspenden sobre la superficie del platillo a una distancia de nanómetros, montados en un brazo articulado (*actuator arm*) que se desplaza radialmente sobre el disco.

El proceso de lectura implica que el cabezal detecta los cambios de magnetización en la superficie del platillo mientras este gira bajo él. Los patrones de magnetización se traducen en señales eléctricas que se decodifican como bits de información. La escritura es el proceso inverso: el cabezal genera un campo magnético que orienta la magnetización de la superficie del platillo en la dirección deseada.

La distancia entre el cabezal y la superficie del platillo es crucial: demasiado cerca provoca un *head crash* (contacto físico entre el cabezal y la superficie), que puede destruir permanentemente los datos. Los HDD modernos incorporan sistemas de carga de aire y resortes de suspensión que mantienen la distancia óptima incluso bajo vibraciones externas.

> Un *head crash* es uno de los fallos más catastróficos de un HDD. Si el cabezal raspa la superficie del platillo, arranca partículas de la capa magnética que pueden causar más daños en cascada. Los datos afectados por un crash suelen ser irrecuperables sin intervención profesional de laboratorio de recuperación de datos.

## Velocidad de rotación (RPM)

La velocidad de rotación de los platillos se mide en revoluciones por minuto (*Revolutions Per Minute*, RPM) y es uno de los factores determinantes del rendimiento del HDD. Los valores estándar son:

- **5,400 RPM:** velocidadd de los discos portátiles y de了一些 aplicaciones de almacenamiento de bajo consumo. Ofrece menor rendimiento y ruido, pero mejor resistencia mecánica y menor consumo energético.
- **7,200 RPM:** la velocidad más común en discos de escritorio y servidores de gama media. Representa el equilibrio entre rendimiento, ruido y vida útil.
- **10,000 RPM:** utilizada en servidores de alta gama donde la latencia de acceso es crítica. Reduce significativamente la latencia pero aumenta el ruido, el calor y el consumo.
- **15,000 RPM:** la velocidad más alta disponible comercialmente, exclusiva de servidores de alta gama. Minimiza la latencia mecánica al máximo posible, pero su uso se ha reducido significativamente con la llegada de los SSD.

A mayor RPM, menor es el tiempo medio de acceso (*Average Seek Time*), ya que el platillo completa una revolución más rápidamente y los datos pasan antes bajo el cabezal. Sin embargo, la mejora del rendimiento no es lineal: saltar de 7,200 a 10,000 RPM (un 39% de aumento) solo reduce la latencia en aproximadamente un 28%, lo que explica por qué los SSD han desplazado a los HDD de alta velocidad en aplicaciones sensibles a la latencia.

## Latencia

La latencia de un HDD es el tiempo que transcurre desde que se solicita un dato hasta que se entrega, y está compuesta por tres componentes principales:

**Latencia de búsqueda (*Seek Time*):** el tiempo necesario para posicionar el brazo actuador sobre la pista correcta del platillo. En discos modernos, el tiempo medio de búsqueda es de 3-8 ms, dependiendo del tipo de disco y la distancia promedio que debe recorrer el brazo.

**Latencia rotacional (*Rotational Latency*):** el tiempo que el platillo tarda en girar hasta que la pista deseada pasa bajo el cabezal. En un disco de 7,200 RPM, una rotación completa toma 8.33 ms, y el tiempo medio de latencia rotacional es la mitad: 4.17 ms.

**Tiempo de transferencia (*Transfer Time*):** el tiempo necesario para leer los datos una vez que el cabezal está posicionado sobre la pista correcta. Este tiempo depende de la densidad lineal de bits y la velocidad de rotación.

La latencia total típica de un HDD oscila entre 5 y 15 milisegundos, dependiendo de la velocidad de rotación y la eficiencia del mecanismo de posicionamiento. Esta latencia es aproximadamente 100-1,000 veces mayor que la de un SSD, lo que explica la dramática diferencia en el rendimiento percibido entre ambos tipos de dispositivo.

## Ventajas y limitaciones

**Ventajas de los HDD:**

- **Coste por gigabyte:** los HDD siguen siendo significativamente más económicos que los SSD por unidad de capacidad. Un HDD de 4 TB puede costar menos que un SSD NVMe de 1 TB.
- **Madurez tecnológica:** más de 60 años de desarrollo han optimizado la tecnología HDD al máximo, resultando en productos fiables y bien comprendidos.
- **Capacidad máxima:** los HDD ofrecen las unidades de mayor capacidad disponibles comercialmente, con modelos de 20-24 TB.
- **Ciclos de escritura ilimitados:** a diferencia de los SSD, los HDD no degradan por escrituras repetidas, ya que la grabación magnética es reversible sin límite práctico.

**Limitaciones de los HDD:**

- **Latencia mecánica:** los componentes móviles introducen una latencia que no puede eliminarse sin eliminar las partes mecánicas.
- **Vulnerabilidad física:** los HDD son sensibles a golpes, vibraciones y caídas durante el funcionamiento. Un impacto durante una operación de escritura puede causar pérdida de datos.
- **Ruido y calor:** la rotación de los platillos y el movimiento del brazo generan ruido y calor que los SSD no producen.
- **Consumo energético:** los HDD consumen más energía que los SSD debido a los motores de rotación y los actuadores mecánicos.
- **Tamaño y peso:** los HDD de 3.5 pulgadas son considerablemente más grandes y pesados que los SSD, limitando su uso en dispositivos compactos.

