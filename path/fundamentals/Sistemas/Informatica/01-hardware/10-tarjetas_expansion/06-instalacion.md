# Instalación y configuración

La instalación de una tarjeta de expansión implica tanto el montaje físico del componente como la configuración del software necesario para su funcionamiento. Aunque el proceso es relativamente sencillo para usuarios con experiencia, existen consideraciones importantes sobre compatibilidad, configuración de BIOS/UEFI y gestión de drivers que pueden afectar al rendimiento y la estabilidad del sistema.

Una instalación correcta va más allá de insertar la tarjeta en el slot: implica verificar la compatibilidad eléctrica y física, configurar los recursos del sistema, instalar los drivers adecuados y verificar que la tarjeta funciona correctamente en el contexto del sistema completo.

## Consideraciones físicas

Antes de la instalación, es necesario verificar varios aspectos físicos del sistema:

**Espacio disponible:** las tarjetas de expansión, especialmente las de gama alta, pueden ser significativamente largas. Una GPU de gama alta puede ocupar 3 slots de expansión y superar los 30 cm de longitud. Antes de adquirir una tarjeta, se debe verificar que el chasis tiene suficiente espacio físico para albergarla, tanto en longitud como en altura.

**Compatibilidad del slot:** verificar que la placa base dispone de un slot PCIe con el número de carriles adecuado para la tarjeta. Una tarjeta x16 requiere un slot x16, pero puede funcionar (a menor velocidad) en slots x8 o x4 si físicamente encaja.

**Alimentación:** algunas tarjetas de expansión de alta gama requieren conectores de alimentación adicionales desde la fuente de alimentación. Las GPU de gama alta pueden requerir uno o dos conectores PCIe de 8 pines, y las tarjetas de red de 10 Gbps pueden necesitar alimentación adicional a través de un conector de placa base o un conector Molex.

**Espacio entre slots:** en placas base con múltiples slots PCIe, es importante verificar que la instalación de una tarjeta no bloquea el acceso a otros slots necesarios. Las GPU de 3 slots pueden impedir el uso de slots PCIe adyacentes, limitando la expansión futura del sistema.

**Flujo de aire:** la instalación de tarjetas de expansión grandes puede obstruir el flujo de aire dentro del chasis, afectando la refrigeración de otros componentes. Es importante considerar el impacto térmico de la nueva tarjeta y ajustar la configuración de ventiladores si es necesario.

## Instalación de *drivers*

Los drivers son los programas que permiten al sistema operativo comunicarse con el hardware. La mayoría de las tarjetas de expansión modernas son reconocidas automáticamente por el sistema operativo (especialmente Windows 10/11), que instala drivers genéricos que proporcionan funcionalidad básica. Sin embargo, para aprovechar todas las prestaciones de la tarjeta, es necesario instalar los drivers específicos del fabricante.

**Drivers de GPU:** los fabricantes NVIDIA, AMD e Intel publican drivers actualizados regularmente que incluyen correcciones de errores, optimizaciones de rendimiento y soporte para nuevos juegos y aplicaciones. La instalación del driver adecuado es crítica para el rendimiento gráfico.

**Drivers de tarjetas de red:** las tarjetas de red suelen requerir drivers específicos del fabricante del chip de red (Intel, Realtek, Broadcom) para alcanzar su velocidad y fiabilidad máximas. Los drivers genéricos del sistema operativo pueden no soportar todas las funciones avanzadas de la tarjeta.

**Drivers de audio:** las tarjetas de sonido requieren drivers especializados que implementan las funcionalidades de procesamiento de audio, control de volumen y gestión de dispositivos de entrada/salida.

**Drivers de captura de vídeo:** las tarjetas de captura necesitan drivers y software de captura que permitan configurar la resolución, tasa de refresco, formato de codificación y destino de la grabación o streaming.

> Los drivers del fabricante suelen ofrecer mejor rendimiento y funcionalidad que los drivers genéricos del sistema operativo, pero pueden ser menos estables en casos raros. Si se experimenta inestabilidad tras instalar un driver del fabricante, se puede recurrir al driver genérico como medida de diagnóstico.

## Gestión en el sistema operativo

Una vez instalada la tarjeta, el sistema operativo gestiona su funcionamiento mediante herramientas específicas:

**Administrador de dispositivos (Windows):** muestra todas las tarjetas instaladas, su estado de funcionamiento y los drivers cargados. Permite actualizar, deshabilitar o desinstalar drivers, y resolver conflictos de recursos (IRQ, direcciones de memoria).

**`lspci` (Linux):** comando que enumera todos los dispositivos PCI detectados por el sistema, incluyendo fabricante, modelo y recursos asignados. Es la herramienta de referencia para diagnosticar problemas de detección de tarjetas.

**Configuración de BIOS/UEFI:** algunos aspectos de las tarjetas de expansión se configuran a nivel de BIOS/UEFI, como la asignación de carriles PCIe, la activación del SR-IOV (*Single Root I/O Virtualization*) para virtualización, o la configuración del modo RAID de los controladores de almacenamiento.

**Gestión de energía:** el sistema operativo gestiona los estados de energía de las tarjetas de expansión, pudiendo reducir su consumo o apagarlos cuando no se utilizan. Para tarjetas de red que deben estar siempre activas, es importante deshabilitar la gestión de energía del dispositivo para evitar interrupciones de conectividad.

