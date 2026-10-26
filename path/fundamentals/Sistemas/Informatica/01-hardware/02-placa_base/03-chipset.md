# Chipset

El chipset es el conjunto de circuitos integrados que controla el flujo de datos entre la CPU, la memoria, los dispositivos de almacenamiento y los puertos de entrada/salida. Es el director de orquesta de la placa base: sin él, ningún componente podría comunicarse con los demás de forma coordinada. El chipset determina qué funcionalidades están disponibles en la placa y cuáles son las limitaciones de expansión y conectividad.

Cada generación de procesadores trae consigo un nuevo chipset diseñado para aprovechar las capacidades del hardware. Un chipset inadecuado puede convertirse en un cuello de botella que impida aprovechar el potencial completo de la CPU y la memoria.

## Qué es y qué controla

El chipset gestiona las comunicaciones de alta velocidad entre los componentes principales del sistema. Sus funciones críticas incluyen:

**Gestión de la memoria:** controla los canales de memoria, la velocidad máxima soportada y el número de módulos DIMM que pueden instalarse. También gestiona la corrección de errores (ECC) en placas que lo soportan.

**Control PCIe:** asigna los carriles PCIe (lanes) entre la CPU y los dispositivos conectados. El número y la distribución de carriles determina cuántas tarjetas gráficas, SSD NVMe y otros dispositivos de alta velocidad pueden conectarse simultáneamente sin perder rendimiento.

**Puertos de almacenamiento:** controla los puertos SATA y M.2, determinando el número de dispositivos de almacenamiento que pueden conectarse y sus velocidades máximas.

**Puertos de E/S:** gestiona los puertos USB, Ethernet, audio y otros puertos integrados en la placa base.

**Gestión energética:** coordina los estados de energía del sistema, incluyendo el modo de reposo, hibernación y las transiciones de voltaje para optimizar el consumo.

> El chipset no es intercambiable. Cada CPU requiere un chipset específico: los procesadores Intel de 13ª generación solo funcionan con chipsets de la serie 600/700, y los AMD Ryzen de serie 7000 requieren chipsets de la serie 600/800. Esta relación exclusiva es una de las razones por las que la elección del procesador determina indirectamente la placa base.

## Evolución de los chipsets

Los chipsets han evolucionado significativamente desde los primeros diseños de los años 80. En los sistemas IBM PC originales, el chipset era un conjunto discreto de circuitos que realizaba funciones básicas de control. Con el tiempo, estas funciones se integraron en chips más compactos y sofisticados.

En los sistemas Intel modernos, la gestión de la memoria y el control PCIe se han trasladado desde el chipset hasta la propia CPU. Esto significa que la CPU controla directamente los carriles PCIe para la GPU y los SSD NVMe, mientras que el chipset gestiona los dispositivos de menor velocidad como puertos SATA, USB y Ethernet. Esta arquitectura descentralizada reduce la latencia para los dispositivos críticos pero complica la asignación de carriles.

AMD adoptó un enfoque similar con su arquitectura Infinity Fabric, que conecta la CPU con el chipset a través de un enlace de alta velocidad. Este enlace, conocido como *PCH* (*Platform Controller Hub*) en Intel o *FCH* (*Fusion Controller Hub*) en AMD, ha reemplazado al antiguo *Northbridge* como el canal principal de comunicación entre la CPU y los dispositivos periféricos.

## Relación con CPU y memoria

La compatibilidad entre chipset y CPU es absoluta: una placa base con un chipset incompatible no podrá ejecutar la CPU, sin importar que el zócalo físico sea compatible. Los fabricantes de procesadores mantienen matrices de compatibilidad que especifican qué chipset funciona con qué generación y modelo de CPU.

El chipset también determina la velocidad máxima de la memoria RAM. No todos los chipsets soportan las frecuencias más altas: un chipset de gama baja puede limitar la memoria a 3200 MHz, mientras que uno de gama alta puede soportar 6000 MHz o más. Esta limitación es relevante para usuarios que buscan maximizar el rendimiento de la memoria.

> La relación entre CPU y chipset evoluciona con cada generación. Lo que era compatible en la generación anterior puede no serlo en la actual, lo que obliga a actualizar la placa base al cambiar de procesador. Esta falta de compatibilidad entre generaciones es una crítica recurrente en la comunidad tecnológica.

## Northbridge y Southbridge (o su equivalente moderno)

Históricamente, los chipsets se dividían en dos componentes principales:

**Northbridge:** controlaba las comunicaciones de alta velocidad entre la CPU, la memoria y la tarjeta gráfica. Era el componente más crítico del chipset y el que más calor generaba. La ubicación del Northbridge cerca del zócalo de CPU no era casualidad: la cercanía física reducía la latencia de las comunicaciones críticas.

**Southbridge:** gestionaba los dispositivos de menor velocidad: puertos SATA, USB, audio, Ethernet y la BIOS/UEFI. El Southbridge se comunicaba con el Northbridge a través de un bus dedicado.

En los sistemas modernos, el Northbridge ha desaparecido como componente separado. Sus funciones han sido integradas directamente en la CPU, lo que ha eliminado un punto potencial de cuello de botella. El Southbridge ha evolucionado hasta convertirse en el *Platform Controller Hub* (PCH) de Intel o el *Fusion Controller Hub* (FCH) de AMD, que sigue gestionando los dispositivos de menor velocidad.

Esta integración ha simplificado el diseño de las placas base pero también ha transferido parte de la complejidad al procesador, que ahora debe gestionar más líneas de señal y más interrupciones.

