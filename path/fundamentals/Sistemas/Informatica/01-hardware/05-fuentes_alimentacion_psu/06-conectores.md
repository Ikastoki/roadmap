# Conectores esenciales

Los conectores de una fuente de alimentación son las interfaces físicas a través de las cuales distribuye energía a los diferentes componentes del sistema. Cada tipo de conector está diseñado para un propósito específico, con un voltaje, una corriente y una forma que garantizan la compatibilidad y la seguridad. Conocer los conectores es esencial para ensamblar un sistema correctamente y evitar errores que puedan dañar los componentes.

## Conector ATX de 24 pines

El conector ATX de 24 pines es el conector principal de alimentación de la placa base. Proporciona los voltajes +3.3V, +5V y +12V necesarios para alimentar la lógica de la placa, los controladores, los puertos USB y otros componentes integrados.

Este conector es el más grande y reconocible de la fuente: un bloque rectangular de 24 pines que se conecta al zócalo correspondiente en la placa base. No es posible conectarlo de forma incorrecta gracias a su diseño asimétrico con pestañas de bloqueo.

Las fuentes más antiguas utilizaban conectores de 20 pines, pero la especificación ATX12V 2.x amplió el conector a 24 pines para proporcionar corriente adicional a los componentes modernos que consumen más energía.

## Conector EPS de 8 pines (CPU)

El conector EPS de 8 pines (*Extended Power Supply*) suministra energía directamente al procesador a través del riel de +12V. Este conector es crítico para alimentar CPUs de alto consumo que pueden requerir más de 200W bajo carga.

Muchas placas base de gama alta incluyen dos conectores EPS de 8 pines para proporcionar corriente adicional a procesadores extremos como los AMD Threadripper o los Intel Xeon. Para la mayoría de los sistemas de consumo, un solo conector EPS de 8 pines es suficiente.

> No confundir el conector EPS de 8 pines con el conector PCIe de 8 pines (6+2). Aunque ambos tienen 8 pines, sus disposiciones de pines son diferentes y no son intercambiables. Conectar un cable PCIe en el zócalo EPS puede causar daños permanentes.

## Conector PCIe de 6+2 pines (GPU)

El conector PCIe de 6+2 pines está diseñado para alimentar tarjetas gráficas dedicadas. Proporciona corriente adicional a través del riel de +12V, necesario para GPU que consumen más de 75W (la capacidad máxima del slot PCIe estándar).

El diseño de 6+2 pines permite una flexibilidad máxima: se puede utilizar como conector de 6 pines para tarjetas gráficas de gama baja o como conector de 8 pines para tarjetas de gama media. Las GPU de alta gama pueden requerir dos o más conectores PCIe de 8 pines, o el nuevo conector de 12 pines (12VHPWR/12V-2x6) introducido con la especificación ATX 3.0.

Las GPU de gama más alta como la NVIDIA RTX 4090 utilizan el conector 12VHPWR, capaz de entregar hasta 600W a través de un solo conector. Este nuevo estándar simplifica la gestión de cables pero requiere una fuente compatible con la especificación ATX 3.0 o un adaptador.

## Conectores SATA

Los conectores SATA de alimentación suministran +5V y +12V a los dispositivos de almacenamiento (discos duros, SSD SATA) y a otros dispositivos de baja potancia como bombas de refrigeración líquida o hubs USB.

Los conectores SATA son conectores de 15 pines con un diseño asimétrico que impide la conexión incorrecta. Cada cable de la fuente suele incluir múltiples conectores SATA en línea, permitiendo alimentar varios dispositivos con un solo cable.

## Conectores Molex

Los conectores Molex (también llamados Berg o peripheral) son conectores legacy de 4 pines que proporcionan +5V y +12V. Originalmente diseñados para alimentar discos duros y dispositivos IDE, hoy en día su uso se ha reducido significativamente.

Los conectores Molex todavía se utilizan para:

- Ventiladores de chasis de 3 pines o 4 pines.
- Bombas de refrigeración líquida.
- Iluminación RGB legacy.
- Algunos dispositivos embebidos.

La mayoría de las fuentes modernas incluyen uno o dos cables con conectores Molex, pero su relevancia sigue disminuyendo a medida que los dispositivos migran a conectores SATA o USB.

