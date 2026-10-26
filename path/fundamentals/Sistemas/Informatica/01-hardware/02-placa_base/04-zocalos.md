# Zócalos y ranuras

Los zócalos y ranuras de una placa base son las interfaces físicas donde se conectan los componentes del sistema. Cada tipo de zócalo está diseñado para un propósito específico y cumple con estándares que garantizan la compatibilidad entre componentes de diferentes fabricantes. La elección de zócalos y ranuras determina qué componentes pueden instalarse y a qué velocidad funcionarán.

## Socket de CPU

El zócalo de CPU (*socket*) es la interfaz física que conecta el procesador con la placa base. No solo proporciona soporte mecánico, sino que establece las conexiones eléctricas necesarias para la transmisión de datos, la distribución de energía y las señales de control.

### Tipos de sockets para CPU

Los fabricantes de procesadores utilizan zócalos propietarios que cambian con cada generación:

**Intel:** los sockets LGA (*Land Grid Array*) colocan los pines de contacto en la placa base, no en el procesador. Los sockets más recientes incluyen LGA 1700 (12ª y 13ª generación), LGA 1851 (Core Ultra) y LGA 4677 (Xeon). Los sockets LGA son delicados: los pines pueden dañarse fácilmente si se manipulan sin cuidado, lo que puede inutilizar la placa base.

**AMD:** los sockets AM4 y AM5 utilizan diferentes enfoques. AM4 es un socket PGA (*Pin Grid Array*) donde los pines están en el procesador, lo que lo hace más resistente a daños accidentales pero más propenso a doblar pines si se cae. AM5 migró a LGA, adoptando el mismo enfoque que Intel.

El zócalo no solo define la compatibilidad física: también determina el número de carriles PCIe disponibles, los canales de memoria soportados y las funcionalidades avanzadas como el overclock.

### Compatibilidad con generaciones de procesadores

Los fabricantes de procesadores cambian de zócalo con relativa frecuencia, lo que limita la capacidad de upgrade. Un usuario con un socket AM4 puede actualizar su procesador dentro de la familia Ryzen sin cambiar la placa base, pero si desea saltar a una generación futura que utilice un zócalo diferente, deberá reemplazar la placa.

Esta estrategia de negocio genera controversia: los usuarios lamentan la falta de compatibilidad a largo plazo, mientras que los fabricantes argumentan que los nuevos zócalos son necesarios para soportar nuevas tecnologías más rápidamente.

> Antes de comprar un procesador, es fundamental verificar la compatibilidad del zócalo y del chipset. Un procesador compatible físicamente con el zócalo puede no funcionar si el chipset no soporta esa generación específica. Las listas de compatibilidad publicadas por los fabricantes de placas base son el recurso más fiable.

## Ranuras RAM

Las ranuras DIMM (*Dual Inline Memory Module*) son los slots donde se insertan los módulos de memoria RAM. El número de ranuras determina la capacidad máxima de memoria y las opciones de configuración de canales.

**DIMM de escritorio:** ranuras de 288 pines para módulos DDR4 o DDR5. La mayoría de placas ATX incluyen cuatro ranuras DIMM, mientras que las Mini-ITX suelen incluir solo dos.

**SODIMM:** ranuras de 260 pines más compactas, utilizadas en portátiles y sistemas Small Form Factor. Los módulos SODIMM son aproximadamente la mitad de grandes que los DIMM estándar.

El tipo de memoria soportado (DDR4 o DDR5) está determinado por la placa base y el chipset. DDR5 ofrece mayores velocidades y mejor eficiencia energética, pero los módulos son más caros y no son retrocompatibles con placas DDR4.

## Ranuras PCIe

PCIe (*Peripheral Component Interconnect Express*) es el estándar para la conexión de tarjetas de expansión de alta velocidad. Los slots PCIe varían en su número de carriles (x1, x4, x8, x16), que determinan el ancho de banda disponible.

**PCIe x16:** el slot principal para tarjetas gráficas. Proporciona 16 carriles de datos, ofreciendo un ancho de banda de hasta 64 GB/s en PCIe 5.0. La mayoría de las placas base incluyen al menos un slot PCIe x16, aunque en placas económicas puede funcionar a x4 o x8 cuando se utilizan múltiples slots.

**PCIe x4:** utilizado para SSD NVMe (en slots adaptadores), tarjetas de red de alta velocidad y controladores de almacenamiento.

**PCIe x1:** para tarjetas con requisitos de ancho de banda modestos como tarjetas de sonido, tarjetas de red básicas y adaptadores USB.

> Los carriles PCIe se comparten entre la CPU y el chipset. Cuando se instalan múltiples dispositivos PCIe, es posible que algunos deban compartir carriles, reduciendo su velocidad. Verificar el diagrama de asignación de carriles de la placa base es esencial antes de instalar múltiples dispositivos de alta velocidad.

## Ranuras M.2

Los slots M.2 son interfaces compactas que permiten la conexión de SSD directamente a la placa base, eliminando la necesidad de cables. Los slots M.2 pueden soportar dispositivos SATA o NVMe (PCIe), dependiendo de la configuración del chipset.

Los M.2 se clasifican por su tipo de llave (*keying*):

- **M-key:** soporta NVMe (PCIe x4) y SATA. Es el tipo más común en placas modernas.
- **B-key:** soporta SATA y PCIe x2. Menos común en la actualidad.
- **B+M key:** compatible con ambos tipos, utilizado en SSD de gama baja.

La longitud de los módulos M.2 también varía (2230, 2242, 2260, 2280), siendo el 2280 (22 mm de ancho, 80 mm de largo) el estándar más común.

## Ranuras de expansión (visión general)

Más allá de PCIe y M.2, las placas base pueden incluir otros tipos de ranuras según su propósito:

**Ranuras de almacenamiento:** los puertos SATA estándar conectan discos duros y SSD de 2.5 y 3.5 pulgadas. El número de puertos SATA varía entre 4 y 8 según el chipset.

**Conectores frontales:** headers para puertos USB frontales, audio HD, botones de encendido y LEDs del chasis.

## Mecanismos de sujeción

Los componentes se sujetan a la placa base mediante mecanismos específicos para cada tipo:

- **CPU:** los sockets LGA utilizan un mecanismo de palanca que presiona el procesador contra los pines.
- **RAM:** los slots DIMM tienen pestañas en los extremos que encajan en las muescas del módulo.
- **GPU:** los slots PCIe utilizan una pestaña de retención en el extremo y un tornillo de fijación en el chasis.
- **M.2:** se fijan con un tornillo o un mecanismo de clip que mantiene el módulo presionado contra el slot.

