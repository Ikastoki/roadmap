# Función de la placa base

La placa base, o tarjeta madre (*motherboard*), es el componente físico que interconecta todos los demás elementos de una computadora. Sin ella, el procesador no tendría cómo comunicarse con la memoria, el almacenamiento, las tarjetas de expansión o los dispositivos de entrada y salida. Es, en términos prácticos, el esqueleto y el sistema nervioso central del ordenador: no solo soporta físicamente los componentes, sino que proporciona los canales eléctricos por los cuales fluye la información.

Comprender la placa base es indispensable para cualquier profesional que desee ensamblar, configurar o mantener sistemas informáticos. Las decisiones de diseño de una placa base —el tipo de chipset, los zócalos disponibles, la calidad de la circuitería— determinan directamente qué componentes son compatibles, qué nivel de rendimiento se puede alcanzar y qué margen de upgrade existe a futuro.

## Historia

Las primeras computadoras no tenían placas base en el sentido moderno. Los cables de conexión estaban montados directamente en bastidores metálicos, y cada conexión era única. La placa base como la conocemos hoy nació con el estándar IBM PC en 1981, que definió un formato estandarizado de tarjeta con zócalos y ranuras que permitían la instalación de componentes de diferentes fabricantes.

El concepto evolucionó rápidamente. Los primeros formatos (AT, Baby AT) dieron paso a ATX en 1995, un diseño que simplificó la distribución de los componentes y mejoró la ventilación. Desde entonces, han surgido variaciones como Micro-ATX, Mini-ITX y E-ATX para adaptarse a diferentes factores de forma, desde servidores compactos hasta estaciones de trabajo de alta gama.

La placa base ha pasado de ser un simple soporte físico a convertirse en un componente sofisticado que integra controladores de red, audio, gráficos integrados y sistemas de gestión energética. Los chipsets modernos gestionan cientos de líneas de señal, y los sistemas de alimentación pueden alcanzar fases de 20+ para alimentar procesadores de alto consumo.

## Rol dentro del computador

La placa base cumple múltiples funciones críticas:

**Interconexión:** proporciona los zócalos para la CPU, la memoria RAM y las tarjetas de expansión, así como los puertos para dispositivos de almacenamiento y E/S. Todos estos componentes se comunican entre sí a través de los buses integrados en la placa.

**Distribución de energía:** recibe la energía de la fuente de alimentación (PSU) y la distribuye a cada componente con el voltaje y la corriente adecuados. Los reguladores de voltaje (VRM) integrados en la placa transforman el voltaje de 12V de la PSU a los niveles más bajos que requieren la CPU y la memoria.

**Gestión del sistema:** el chipset de la placa base controla el flujo de datos entre la CPU, la memoria y los dispositivos periféricos. También gestiona las interrupciones, los temporizadores y otros eventos del sistema.

**Firmware:** la BIOS o UEFI almacena en la placa base la configuración básica del sistema y el código necesario para inicializar los componentes durante el arranque.

> Una placa base de gama alta no necesariamente ofrece mejor rendimiento en tareas básicas que una de gama media. Las diferencias principales radican en la calidad de los componentes (VRM, condensadores), el número de ranuras de expansión, las opciones de overclock y la durabilidad a largo plazo.

## Relación con otros componentes

La placa base es el punto de encuentro de prácticamente todos los componentes del sistema:

- **CPU:** se inserta en el zócalo (socket) correspondiente, que define la compatibilidad con fabricantes (Intel o AMD) y generaciones específicas.
- **Memoria RAM:** se conecta a los slots DIMM o SODIMM, cuyo tipo (DDR4, DDR5) y número determinan la capacidad y velocidad máxima de memoria.
- **GPU:** se inserta en las ranuras PCIe x16, que proporcionan el ancho de banda necesario para la transferencia de datos gráficos.
- **Almacenamiento:** los puertos SATA y los slots M.2 conectan discos duros y SSD al sistema.
- **Dispositivos de E/S:** los puertos USB, audio, red y video integrados permiten la conexión de periféricos sin necesidad de tarjetas adicionales.
- **Fuente de alimentación:** los conectores ATX de 24 pines y EPS de 8 pines suministran energía a la placa y a la CPU.

Cada uno de estos componentes interactúa con la placa base a través de interfaces estandarizadas, lo que permite mezclar componentes de diferentes fabricantes siempre que respeten los estándares correspondientes.

