# Refrigeración por aire

La refrigeración por aire es el método más común, económico y sencillo de disipar el calor en un sistema informático. Utiliza disipadores metálicos que absorben el calor del componente y ventiladores que fuerzan aire a través de las aletas del disipador, transfiriendo la energía térmica al aire circundante. La eficiencia de este sistema depende de la calidad del disipador, el caudal de aire de los ventiladores, la temperatura ambiente y el flujo de aire general del chasis.

A pesar de su aparente simplicidad, la refrigeración por aire alcanza niveles de rendimiento que rivalizan con muchas soluciones de refrigeración líquida para cargas de trabajo moderadas. Un buen disipador de torre con ventiladores de calidad puede mantener un procesador de 125W de TDP por debajo de 80°C incluso bajo carga continua, a un coste fracción de una solución líquida equivalente.

## Disipadores (materiales, aletas, heatpipes)

El disipador es la estructura metálica que absorbe el calor del componente y lo distribuye para su disipación al aire. Los disipadores modernos combinan múltiples materiales y diseño para maximizar la transferencia de calor:

**Materiales:** el aluminio es el material más utilizado por su excelente relación conductividad térmica/peso/coste (237 W/mK). El cobre ofrece mayor conductividad (401 W/mK) pero es más pesado y caro, por lo que se utiliza típicamente en la base de contacto con el procesador, mientras que las aletas son de aluminio. Algunos disipadores de alta gama combinan base de cobre con aletas de aluminio para optimizar ambos parámetros.

**Aletas:** son las láminas metálicas que aumentan enormemente la superficie de contacto con el aire. Un disipador típico puede tener 50-100 aletas con espaciado de 1-2 mm. El diseño, número y espaciado de las aletas determinan la capacidad de disipación del calor: más aletas significan mayor superficie pero también mayor resistencia al flujo de aire.

**Heatpipes (*tubos de calor*):** son tubos de cobre sellados que contienen un líquido en su interior (típicamente agua en condiciones de baja presión). El calor del componente vaporiza el líquido en la base del heatpipe, el vapor se desplaza hasta la zona fría (las aletas del disipador), se condensa liberando el calor, y el líquido regresa por gravedad o por acción capilar al ciclo. Los heatpipes permiten transferir el calor desde la base del disipador hasta las zonas más alejadas con alta eficiencia, distribuyendo el calor uniformemente sobre todas las aletas.

La cantidad y calidad de los heatpipes es un indicador fiable de la capacidad de un disipador. Un disipador con 4 heatpipes de 6 mm puede gestionar 150-200W de TDP, mientras que uno con 6 heatpipes de 8 mm puede superar los 250W.

> El sellado de los heatpipes es crítico: si la unión se degrada y el líquido se evapora, el heatpipe pierde completamente su capacidad de transferencia de calor. Un heatpipe defectuoso se reconoce porque la zona de las aletas cercana al heatpipe permanece fría mientras la base del disipador está caliente.

## Tipos de ventiladores (tamaño, CFM, ruido)

Los ventiladores son los componentes activos que fuerzan el flujo de aire a través del disipador y del chasis. Sus especificaciones principales son:

**Tamaño:** los ventiladores de PC se clasifican por su diámetro en milímetros. Los más comunes son 80 mm, 120 mm y 140 mm. A mayor tamaño, menor velocidad necesaria para mover la misma cantidad de aire, lo que se traduce en menos ruido. Los ventiladores de 120 mm son el estándar actual para la mayoría de configuraciones, mientras que los de 140 mm se utilizan en disipadores de alta gama y chasis silenciosos.

**Caudal de aire (CFM):** *Cubic Feet per Minute*, mide el volumen de aire que el ventilador puede mover por minuto. Un ventilador de 120 mm puede mover entre 30-80 CFM dependiendo de su diseño. Un CFM más alto significa mejor refrigeración, pero también generalmente más ruido.

**Presión estática (mmH2O):** mide la fuerza con que el ventilador empuja el aire a través de obstáculos como las aletas de un disipador o los filtros de polvo. Los ventiladores de alta presión son ideales para disipadores de torre y radiadores, donde el aire debe superar la resistencia de las aletas.

**Ruido (dBA):** los ventiladores de PC típicos operan entre 20-40 dBA. Por encima de 35 dBA, el ruido se vuelve noticeable en un entorno silencioso. Los ventiladores de gama silenciosa priorizan la reducción de ruido a costa de un CFM ligeramente menor, mientras que los de alto rendimiento priorizan el caudal de aire.

**Velocidad de rotación (RPM):** *Revolutions Per Minute*, determina cuántas veces gira el ventilador por minuto. La mayoría de los ventiladores modernos son PWM (*Pulse Width Modulation*), lo que permite al sistema ajustar la velocidad entre 0-100% según la temperatura, equilibrando refrigeración y ruido.

