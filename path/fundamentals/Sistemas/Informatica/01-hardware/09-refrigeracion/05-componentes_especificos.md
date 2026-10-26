# Refrigeración de componentes específicos

Aunque el procesador y la tarjeta gráfica son los componentes que generan más calor y reciben la mayor atención en términos de refrigeración, existen otros componentes del sistema que también requieren gestión térmica adecuada. El sobrecalentamiento de estos componentes secundarios puede provocar inestabilidad, degradación del rendimiento o incluso fallos que afecten a todo el sistema.

La refrigeración de componentes específicos es frecuentemente overlooked por los usuarios principiantes, que centran toda su atención en el disipador de la CPU. Sin embargo, un SSD NVMe que alcanza temperaturas de 80°C reduce drásticamente su velocidad, y unos VRM de la placa base que superan los 110°C pueden provocar reinicios inesperados o daños permanentes.

## VRM de la placa base

Los VRM (*Voltage Regulator Module*) son los circuitos de regulación de voltaje de la placa base que convierten la energía de la fuente de alimentación en el voltaje preciso y estable que necesita el procesador. Los VRM generan una cantidad significativa de calor como subproduct de su operación, y su temperatura afecta directamente la estabilidad del sistema.

En placas base de gama alta, los VRM están equipados con disipadores pasivos de aluminio que cubren los MOSFET y los condensadores. En configuraciones de alto consumo (procesadores de más de 200W o con overclock activo), estos disipadores pasivos pueden ser insuficientes, provocando que los VRM alcancen temperaturas que reducen su eficiencia y lifespan.

La refrigeración activa de los VRM se logra mediante:

- **Flujo de aire del ventilador de la CPU:** un disipador de torre con ventilador que dirige aire hacia los VRM circundantes puede reducir significativamente su temperatura.
- **Ventiladores de chasis bien posicionados:** un ventilador frontal que proporcione aire fresco constante a la zona de los VRM mejora su refrigeración.
- **Disipadores VRM aftermarket:** disipadores de aluminio con aletas que se adhieren a los VRM para aumentar su superficie de disipación.

## SSD NVMe

Los SSD NVMe de alta gama pueden alcanzar temperaturas de 70-80°C bajo carga sostenida de lectura/escritura, lo que provoca la activación del *thermal throttling*: el SSD reduce automáticamente su velocidad para evitar daños por calor. Un SSD NVMe que experimenta throttling puede perder más del 50% de su rendimiento, pasando de 7,000 MB/s a menos de 2,000 MB/s.

La refrigeración de los SSD NVMe se logra mediante:

- **Disipadores incluidos:** muchos SSD NVMe de gama alta incluyen disipadores de aluminio o cobre que aumentan significativamente la superficie de disipación. Algunos SSD se venden sin disipador, asumiendo que el usuario instalará uno por separado.
- **Disipadores de la placa base:** la mayoría de placas base modernas incluyen disipadores de aluminio en los slots M.2 que cubren los SSD NVMe. Estos disipadores suelen ser suficientes para SSD de gama media.
- **Disipadores aftermarket:** los disipadores de cobre o aluminio de terceros pueden reducir la temperatura del SSD en 10-20°C, eliminando el throttling en la mayoría de situaciones.

> La temperatura del SSD NVMe afecta no solo al rendimiento sino también a la durabilidad. Las celdas NAND se degradan más rápidamente a temperaturas elevadas, reduciendo la vida útil del dispositivo. Mantener el SSD por debajo de 60°C maximiza tanto el rendimiento como la longevidad.

## VRAM de la GPU

La memoria VRAM de las tarjetas gráficas de alta gama genera cantidades significativas de calor, particularmente las GDDR6X que operan a altas frecuencias y voltajes. La refrigeración de la VRAM depende del diseño del sistema de refrigeración de la GPU:

**Cooler de referencia (*blower*):** los disipadores de referencia de las GPU utilizan un diseño de flujo de aire que extrae aire del chasis y lo expulsa por la trasera de la tarjeta. La refrigeración de la VRAM depende del contacto directo con el disipador y del flujo de aire general.

**Cooler de terceros (*aftermarket*):** los fabricantes como ASUS, MSI y Gigabyte diseñan sistemas de refrigeración personalizados con tubos de calor que contactan directamente con la VRAM y los VRM, proporcionando refrigeración significativamente superior a los diseños de referencia.

**Waterblock para GPU:** las soluciones de refrigeración líquida para GPU incluyen bloques de agua que cubren tanto el chip gráfico como la VRAM y los VRM, proporcionando la refrigeración más eficiente posible para todos los componentes de la tarjeta.

