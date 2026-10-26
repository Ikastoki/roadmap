# Pasta térmica y almohadillas

La interfaz térmica (*Thermal Interface Material*, TIM) es uno de los componentes más pequeños pero más críticos en todo sistema de refrigeración. Su función es eliminar los vacíos microscópicos entre la superficie del procesador y la base del disipador, que de otro modo estarían llenos de aire —un aislante térmico terrible con una conductividad de apenas 0.026 W/mK. La pasta térmica, con una conductividad típica de 5-15 W/mK, mejora drásticamente la transferencia de calor en esta interfaz.

La importancia de la pasta térmica es frecuentemente subestimada. Una aplicación deficiente —demasiada pasta, muy poca pasta, o una distribución irregular— puede provocar temperaturas 5-15°C superiores a las óptimas, lo que se traduce en throttling, inestabilidad o acortamiento de la vida útil del procesador.

## La importancia de la interfaz térmica (TIM)

La superficie del *heat spreader* del procesador y la base del disipador, aunque parezcan perfectamente planas a simple vista, presentan irregularidades microscópicas que impiden el contacto directo completo entre ambas superficies. Sin una interfaz térmica, estos vacíos se llenarían de aire, creando barreras aislantes que impedirían la transferencia eficiente de calor.

La función de la pasta térmica es rellenar estas imperfecciones con un material que tenga una conductividad térmica significativamente superior a la del aire. Aunque la pasta térmica es un mal conductor comparada con el cobre o el aluminio, su capacidad para eliminar los vacíos de aire la convierte en un componente indispensable.

Las almohadillas térmicas cumplen la misma función pero en forma de lámina preformada. Se utilizan en componentes donde la presión de contacto es constante y el espesor de la interfaz es conocido, como los módulos de memoria VRAM de las GPU o los chips del controlador en las tarjetas gráficas.

> La pasta térmica no mejora la refrigeración por sí misma: la mejora proviene de la eliminación de los vacíos de aire. Una aplicación perfectamente plana de las superficies metálicas sin pasta térmica sería ideal, pero la fabricación de superficies a nivel atómico es económicamente inviable, haciendo de la pasta térmica una solución práctica y efectiva.

## Tipos y técnicas de aplicación

**Pasta térmica de silicona:** la más económica y común, con conductividad de 1-5 W/mK. Adecuada para sistemas de gama básica donde la disipación no es crítica.

**Pasta térmica de base metálica:** utiliza partículas de metales como plata, cobre o aluminio suspendidas en una base de silicona, alcanzando conductividades de 5-15 W/mK. Es el estándar para la mayoría de configuraciones de gama media.

**Pasta térmica líquida metal:** utiliza galio o aleaciones de galio como base, alcanzando conductividades de 20-80 W/mK. Son las más eficientes pero también las más difíciles de aplicar y potencialmente conductoras eléctricamente, lo que requiere precaución para evitar cortocircuitos.

**Pasta térmica de base de carbono (*carbon-based*):** utiliza grafeno o nanotubos de carbono, ofreciendo buena conductividad (8-15 W/mK) con la ventaja de no ser eléctricamente conductiva.

Las técnicas de aplicación más comunes son:

- **Punto central:** una gota del tamaño de un guisante en el centro del procesador. La presión del disipador distribuye la pasta de forma irregular pero generalmente aceptable. Es la técnica más simple y recomendada para la mayoría de usuarios.
- **Cruz:** trazar una cruz con la pasta sobre el procesador. Distribuye la pasta de forma más uniforme pero es más difícil de controlar la cantidad exacta.
- **Cinco puntos:** colocar una gota pequeña en cada esquina y una en el centro. Proporciona la distribución más uniforme pero es la más laboriosa.

La cantidad óptima de pasta térmica es la mínima necesaria para cubrir toda la superficie del procesador sin excederse. Demasiada pasta puede contaminar los componentes circundantes de la placa base, mientras que muy poca deja áreas sin cubrir que actúan como puntos calientes.

