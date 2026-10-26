# Unidades de estado sólido (SSD)

El SSD (*Solid State Drive*) ha transformado radicalmente el rendimiento de los sistemas informáticos desde su adopción generalizada a finales de la década de 2010. Al eliminar por completo las partes mecánicas y basarse en memoria flash, los SSD ofrecen latencias, velocidades de transferencia y resistencia física que los HDD no pueden igualar. Un sistema con SSD como unidad de sistema experimenta una mejora de rendimiento perceptible en prácticamente todas las operaciones cotidianas: arranque, carga de aplicaciones, apertura de archivos y multitarea.

Sin embargo, no todos los SSD son iguales. Las diferencias en la tecnología de celdas NAND, los formatos físicos y las interfaces de conexión crean un espectro amplio de rendimiento y durabilidad que el usuario debe comprender para elegir el dispositivo adecuado. Este artículo analiza los principios tecnológicos, los formatos disponibles y las diferencias de rendimiento entre las distintas opciones de SSD.

## Principios de la memoria Flash (NAND)

La memoria NAND es la base de practically todos los SSD modernos. A diferencia de la memoria DRAM de la RAM, que almacena cada bit en un condensador, la NAND almacena bits en celdas formadas por transistores flotantes o capas de*carga trapping*. Estos transistores pueden retener carga eléctrica durante años sin energía, proporcionando la no volatilidad necesaria para un dispositivo de almacenamiento.

Cada celda de NAND puede almacenar uno o más bits de información, dependiendo de cuántos niveles de carga se utilicen para representar los datos:

- **SLC (*Single-Level Cell*):** almacena 1 bit por celda (2 niveles de carga). Es la variante más rápida, más resistente y más cara. Ofrece aproximadamente 100,000 ciclos de escritura por celda. Se utiliza en aplicaciones industriales y embebidas donde la fiabilidad es crítica.

- **MLC (*Multi-Level Cell*):** almacena 2 bits por celda (4 niveles de carga). Ofrece un buen equilibrio entre capacidad, rendimiento y coste. Con aproximadamente 10,000 ciclos de escritura por celda, es adecuada para uso general de consumo.

- **TLC (*Triple-Level Cell*):** almacena 3 bits por celda (8 niveles de carga). Es la tecnología dominante en los SSD de consumo actuales, ofreciendo la mejor relación capacidad/precio. La resistencia a la escritura se reduce a aproximadamente 1,000-3,000 ciclos por celda, pero los controladores modernos y la corrección de errores compensan esta limitación.

- **QLC (*Quad-Level Cell*):** almacena 4 bits por celda (16 niveles de carga). Maximiza la densidad y reduce el coste por gigabyte, pero sacrifica velocidad de escritura y resistencia (aproximadamente 100-1,000 ciclos por celda). Es ideal para almacenamiento de lectura frecuente donde la escritura es infrecuente.

> La tendencia hacia mayor densidad de bits por celda (de SLC a QLC) es un trade-off fundamental: más bits por celda significan mayor capacidad y menor coste, pero también menor velocidad, menor resistencia a la escritura y mayor complejidad en el controlador para gestionar los múltiples niveles de carga con precisión.

## Formatos de SSD

Los SSD se comercializan en múltiples formatos físicos, cada uno diseñado para un escenario de uso específico:

**2.5 pulgadas SATA:** el formato más tradicional, con el mismo factor de forma que los discos duros portátiles. Utiliza la interfaz SATA III (6 Gbps) y ofrece velocidades de hasta 550 MB/s de lectura y 520 MB/s de escritura. Es la opción más compatible con sistemas antiguos, ya que encaja en los bahías estándar de 2.5 pulgadas sin necesidad de adaptadores.

**M.2 SATA:** un formato compacto que se conecta directamente a la placa base mediante el slot M.2. Utiliza la misma interfaz SATA que los SSD de 2.5 pulgadas, por lo que su rendimiento máximo es idéntico (550/520 MB/s), pero su factor de forma diminuto (22 mm de ancho, 80 mm de largo en el estándar 2280) lo hace ideal para portátiles y sistemas compactos.

**M.2 NVMe:** el mismo factor de forma M.2 pero con interfaz NVMe sobre PCIe, que ofrece un ancho de banda significativamente mayor. Los SSD M.2 NVMe de PCIe 3.0 alcanzan 3,500 MB/s, los de PCIe 4.0 superan los 7,000 MB/s y los más recientes de PCIe 5.0 superan los 12,000 MB/s. Este formato es actualmente el estándar de referencia para rendimiento.

**AIC (*Add-in-Card*):** SSD en formato de tarjeta de expansión que se inserta en un slot PCIe. Ofrece rendimiento similar a los M.2 NVMe pero en un formato más grande que permite mejores disipadores térmicos. Se utiliza en servidores y estaciones de trabajo donde el calor generado por SSDs NVMe de alta gama puede provocar *thermal throttling*.

**U.2 y U.3:** formatos de 2.5 pulgadas pero con interfaz NVMe, diseñados para servidores. Combina la compatibilidad física con bahías estándar de 2.5 pulgadas con el rendimiento de NVMe. La versión U.3 añade compatibilidad retroactiva con dispositivos SATA.

## Interfaces de conexión

La interfaz de conexión determina el ancho de banda máximo que el SSD puede utilizar para comunicarse con el procesador:

**SATA III (6 Gbps):** la interfaz más extendida, con un ancho de banda teórico de 600 MB/s (550-560 MB/s prácticos). Es la opción más compatible pero también la más limitada en rendimiento. Adecuada para uso general y para sistemas donde la compatibilidad con hardware antiguo es prioritaria.

**NVMe sobre PCIe:** la interfaz de referencia para SSD de alto rendimiento. Utiliza los carriles PCIe directamente conectados al procesador, eliminando el cuello de botella del controlador SATA. PCIe 3.0 ofrece 3.5 GB/s por carril (típicamente x4 = 14 GB/s teóricos), PCIe 4.0 dobla esa cifra a 7 GB/s por carril, y PCIe 5.0 la vuelve a doblar a 14 GB/s por carril.

**NVMe sobre PCIe con múltiples carriles:** algunos SSD de servidor utilizan x8 o x16 carriles PCIe para alcanzar anchos de banda aún mayores, superando los 30 GB/s en configuraciones x16 de PCIe 4.0.

La elección de interfaz tiene un impacto directo en el rendimiento: un SSD SATA de 550 MB/s es 6 veces más lento que un SSD NVMe PCIe 4.0 de 7,000 MB/s. Sin embargo, para la mayoría de las tareas cotidianas, la diferencia perceptible es menor, ya que el cuello de botella suele estar en la latencia de acceso más que en el ancho de banda bruto.

## Almacenamiento híbrido

El almacenamiento híbrido combina un HDD de gran capacidad con un SSD de menor tamaño en un solo paquete, gestionado por firmware que mueve automáticamente los datos más accedidos al SSD y relega los menos usados al HDD. Los dispositivos *SSHD* (*Solid State Hybrid Drive*) incorporan esta lógica de forma transparente para el sistema operativo.

Aunque la concepto es atractivo, los SSD híbridos nunca alcanzaron la popularidad esperada. La gestión automática de datos es menos eficiente que una configuración explícita de SSD + HDD bajo control del usuario, y la diferencia de precio con un SSD puro de capacidad comparable disminuyó rápidamente. Hoy en día, la configuración de un SSD como unidad de sistema y un HDD como almacenamiento secundario es casi universalmente preferida sobre los SSDH.

