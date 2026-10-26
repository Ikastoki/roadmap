# Refrigeración líquida

La refrigeración líquida utiliza un fluido —típicamente agua o una mezcla de agua y glicol— como medio de transferencia de calor en lugar del aire. El líquido tiene una capacidad calorífica significativamente mayor que el aire (4.186 J/gK frente a 1.005 J/gK), lo que le permite absorber y transportar más calor por unidad de volumen. Esta propiedad fundamental permite a los sistemas de refrigeración líquida gestionar cargas térmicas más altas con menos ruido que las soluciones de aire equivalentes.

La refrigeración líquida no es inherentemente superior a la refrigeración por aire: un sistema de aire bien diseñado puede igualar o superar a un sistema líquido mal diseñado. La ventaja del líquido radica en su capacidad para transferir el calor desde el componente hasta una zona donde pueda disiparse de forma más eficiente, como un radiador grande colocado en una zona de alta corriente de aire.

## Sistemas AIO (*All-In-One*)

Los sistemas AIO son soluciones de refrigeración líquida selladas de fábrica, listas para instalar sin mantenimiento. Consisten en un bloque de agua que se monta sobre el procesador, una bomba integrada, un radiador con ventiladores y tubos de conexión prerellenos con refrigerante.

Las principales ventajas de los AIO son la facilidad de instalación (no requieren conocimientos de fontanería ni mantenimiento de circuitos) y la garantía de que el sistema está correctamente sellado y llenado. Los AIO vienen en diferentes tamaños de radiador: 120 mm, 240 mm, 280 mm y 360 mm, donde el número indica el tamaño del radiador y la cantidad de ventiladores que puede albergar.

Un AIO de 240 mm puede gestionar procesadores con TDP de hasta 200W, mientras que uno de 360 mm puede manejar más de 300W, acercándose al rendimiento de las soluciones personalizadas. La desventaja de los AIO es que, al ser sellados, no permiten la expansión ni el reemplazo de componentes individuales. Si la bomba falla o el refrigerante se evapora con el tiempo, el sistema completo debe reemplazarse.

> Los AIO modernos han mejorado significativamente su fiabilidad respecto a las primeras generaciones. Los fabricantes como Corsair, NZXT y Arctic ofrecen garantías de 5-6 años, y las tasas de fallo han disminuido considerablemente. Sin embargo, la expectativa de vida útil de un AIO es típicamente de 5-7 años, frente a los 10+ años de un buen disipador de aire.

## *Custom Loop*

Un *custom loop* es un sistema de refrigeración líquida diseñado y ensamblado por el usuario, utilizando componentes individuales seleccionados según las necesidades específicas. Los componentes principales son:

**Bloque de agua (*water block*):** el componente que entra en contacto directo con el procesador (o GPU). Los bloques de cobre ofrecen la mejor transferencia térmica, mientras que los de acero inoxidable son más duraderos y resistentes a la corrosión.

**Bomba:** el corazón del sistema, encargada de circular el refrigerante a través del circuito. Las bombas D5 y DDC son los estándares de la industria, con diferentes variantes de caudal y presión. La bomba puede integrarse en el bloque de agua (*pump-on-block*) o montarse de forma independiente en el circuito.

**Radiador:** el componente que disipa el calor del refrigerante al aire. Los radiadores de aluminio son ligeros y económicos, mientras que los de cobre ofrecen mejor transferencia térmica pero son más pesados y caros. El grosor del radiador (25-60 mm) y el número de filas de tubos afectan significativamente su capacidad de disipación.

**Tubos:** conectan todos los componentes del circuito. Los tubos blandos (*soft tubing*) de silicona o PVC son fáciles de manipular pero menos estéticos. Los tubos duros (*hard tubing*) de acrílico o PETG ofrecen una estética superior pero requieren habilidad y herramientas especiales para doblar y cortar.

**Refrigerante:** el fluido que transporta el calor. Las mezclas de agua con glicol y aditivos anticorrosivos son el estándar, ya que proporcionan protección contra la congelación, la corrosión y el crecimiento de microorganismos. Los refrigerantes de colores son populares por estética, pero algunos aditivos de color pueden degradarse con el tiempo y obstruir el circuito.

> Un custom loop bien diseñado puede alcanzar temperaturas 5-15°C inferiores a un AIO equivalente, gracias a radiadores más grandes, mejor contacto térmico y la posibilidad de refrigerar tanto CPU como GPU en el mismo circuito. Sin embargo, el coste, la complejidad de mantenimiento y el riesgo de fugas lo convierten en una opción para entusiastas experimentados.

## Refrigeración por inmersión y extremos

**Refrigeración por inmersión:** sumerge directamente los componentes electrónicos en un líquido dieléctrico (no conductivo eléctrico) que absorbe el calor directamente de los chips. Existen dos tipos:

- **Inmersión en fase única:** los componentes se sumergen en un líquido dieléctrico que circula por el circuito y disipa el calor en un radiador externo. Es utilizada en centros de datos de alta densidad donde la refrigeración por aire es insuficiente.

- **Inmersión en dos fases:** el líquido hierve en contacto con los componentes calientes, absorbiendo enormes cantidades de calor mediante el cambio de fase (vaporización). El vapor sube, se condensa en un condensador y regresa al tanque. Es extremadamente eficiente pero compleja de implementar.

**Aceite mineral:** algunos entusiastas experimentan con la inmersión de componentes en aceite mineral, que es dieléctrico y no conductivo. Aunque funcional, el aceite es viscoso, difficulta el mantenimiento y puede degradar ciertos materiales plásticos con el tiempo.

**Refrigeración termoeléctrica (Peltier):** utiliza el efecto Peltier —una diferencia de temperatura que se crea cuando la corriente eléctrica fluye a través de la unión de dos conductores diferentes— para enfriar activamente el procesador. Los dispositivos Peltier pueden alcanzar temperaturas bajo cero, pero consumen mucha energía y generan una cantidad igual de calor en su lado caliente, que debe ser gestionada por un sistema de refrigeración convencional.

