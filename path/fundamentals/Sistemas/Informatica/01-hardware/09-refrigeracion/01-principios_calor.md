# Principios de la transferencia de calor

Toda la electrónica de un sistema informático genera calor como subproducto inevitable del paso de corriente eléctrica a través de los componentes. Este calor, si no se gestiona adecuadamente, degrada el rendimiento, acorta la vida útil de los componentes y puede provocar fallos catastróficos. La refrigeración no es un complemento opcional sino un requisito fundamental para el funcionamiento seguro y eficiente de cualquier sistema.

Comprender los principios físicos de la transferencia de calor permite diseñar sistemas de refrigeración más eficientes, diagnosticar problemas térmicos y tomar decisiones informadas sobre la configuración de ventilación, pasta térmica y componentes. Los tres mecanismos de transferencia de calor —conducción, convección y radiación— operan simultáneamente en todo sistema informático, y el diseño de refrigeración óptimo aprovecha los tres de forma coordinada.

## Importancia de la refrigeración

Los componentes electrónicos operan dentro de rangos de temperatura específicos definidos por los fabricantes. La CPU típica puede funcionar de 0°C a 100°C, pero su vida útil y rendimiento óptimo se encuentran en el rango de 40-80°C. Superar la temperatura máxima operativa provoca la activación de mecanismos de protección que reducen el rendimiento o apagan el sistema para evitar daños permanentes.

Una refrigeración inadecuada tiene consecuencias inmediatas y a largo plazo. A corto plazo, el sistema experimenta *thermal throttling* —reducción automática de la velocidad del procesador para mantener la temperatura dentro de límites seguros— que se traduce en un rendimiento significativamente inferior al esperado. A largo plazo, la exposición continua a temperaturas elevadas acelera la degradación de los componentes electrónicos, particularmente los condensadores, reduciendo la vida útil del sistema.

> La temperatura ambiente del local donde se encuentra el sistema afecta directamente la eficiencia de la refrigeración. Un sistema que opera en un despacho a 25°C tiene un margen térmico mucho más amplio que uno en un trastero a 40°C, donde la refrigeración debe trabajar significativamente más para mantener los componentes dentro de rango.

## Calor y rendimiento

La relación entre temperatura y rendimiento es inversamente proporcional: a mayor temperatura, menor rendimiento. Este fenómeno tiene múltiples manifestaciones:

**Reducción de frecuencia:** los procesadores modernos ajustan dinámicamente su frecuencia en función de la temperatura disponible. Un procesador que puede alcanzar 5.0 GHz a 70°C puede limitarse a 4.5 GHz a 90°C, perdiendo aproximadamente un 10% de rendimiento.

**Errores de memoria:** la DRAM de la memoria RAM es sensible a la temperatura. A temperaturas elevadas, las celdas de memoria pueden retener la carga durante menos tiempo, provocando errores bit a bit que afectan a la integridad de los datos.

**Acortamiento de vida útil:** la regla de Arrhenius establece que la tasa de degradación química de un componente se duplica por cada 10°C de aumento de temperatura. Un condensador que dura 10 años a 50°C puede durar solo 5 años a 60°C.

**Inestabilidad del sistema:** las fluctuaciones de temperatura provocan expansiones y contracciones térmicas que pueden aflojar contactos eléctricos, causando bloqueos aleatorios y reinicios inesperados.

## Thermal throttling

El *thermal throttling* es el mecanismo de protección que reducen la velocidad del procesador cuando alcanza su temperatura máxima operativa. La CPU monitoriza continuamente su temperatura mediante sensores integrados, y cuando se alcanza el límite (típicamente 90-100°C para procesadores de escritorio), el controlador de energía reduce la frecuencia y el voltaje para disminuir la generación de calor.

El thermal throttling es una protección necesaria, pero su activación indica que el sistema de refrigeración es insuficiente para la carga de trabajo. Un sistema que experimenta throttling frecuente no está entregando el rendimiento que sus componentes son capaces de proporcionar.

## Conducción, convección, radiación

**Conducción:** es la transferencia de calor a través del contacto directo entre materiales. En un sistema informático, la conducción ocurre entre el die del procesador y el disipador a través de la pasta térmica, y entre los componentes del disipador y sus aletas. La eficiencia de la conducción depende de la conductividad térmica del material y del área de contacto. Por eso la pasta térmica —con conductividad típica de 5-15 W/mK— es tan importante: rellena las imperfecciones microscópicas entre el procesador y el disipador, maximizando el contacto efectivo.

**Convección:** es la transferencia de calor mediante el movimiento de un fluido (aire o líquido) sobre una superficie caliente. Es el mecanismo principal de disipación en los sistemas de refrigeración por aire: el ventilador fuerza aire a pasar sobre las aletas del disipador, que transfieren el calor del metal al aire. La eficiencia de la convección depende de la velocidad del flujo de aire, la temperatura del aire de entrada y la superficie de contacto entre el aire y el disipador.

**Radiación:** es la emisión de energía térmica en forma de ondas electromagnéticas. Aunque presente en todo objeto con temperatura superior al cero absoluto, la radiación es el mecanismo menos significativo en los sistemas informáticos convencionales. Solo se vuelve relevante en entornos de vacío o en sistemas de refrigeración por inmersión donde la conducción y la convección no están disponibles.

> La pasta térmica no es un buen conductor térmico comparada con el cobre o el aluminio: su conductividad es 100-300 veces menor. Su función no es conducir el calor sino eliminar los vacíos de aire entre dos superficies metálicas, que son un aislante térmico significativamente peor que la propia pasta.

## El concepto de TDP (Thermal Design Power)

El TDP (*Thermal Design Power*) es la cantidad máxima de calor que el sistema de refrigeración debe ser capaz de disipar para mantener al procesador dentro de su rango de temperatura operativo bajo carga continua. Se mide en vatios (W) y se especifica por el fabricante del procesador.

Un procesador con un TDP de 65W genera un máximo de 65 vatios de calor bajo carga normal. Un procesador con TDP de 125W genera hasta 125W de calor. La elección del disipador y los ventiladores debe garantizar que el sistema de refrigeración pueda gestionar al menos el TDP del procesador, idealmente con un margen de seguridad.

> El TDP no representa el consumo máximo del procesador. Los procesadores modernos pueden superar el TDP durante ráfagas breves de boost (*PL2* en Intel, *PPT* en AMD), generando calor significativamente superior al TDP nominal durante períodos cortos. Los sistemas de refrigeración deben tener capacidad para gestionar estos picos, no solo el TDP base.

