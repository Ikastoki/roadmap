# Circuitería y calidad eléctrica

La calidad eléctrica de una placa base es uno de los factores más determinantes de la estabilidad y longevidad de un sistema informático. Un diseño eléctrico deficiente puede provocar bloqueos aleatorios, reinicios inesperados, degradación gradual de los componentes e incluso daños permanentes. La circuitería de una placa base comprende los reguladores de voltaje, los condensadores, los mosfets, las bobinas y los circuitos de protección que garantizan que cada componente reciba la energía limpia y estable que necesita.

Comprender la circuitería es especialmente relevante para usuarios que buscan overclock, ya que los ajustes de voltaje y frecuencia exigen una fuente de alimentación estable que solo una buena circuitería puede proporcionar. También es importante para evaluar la durabilidad de una placa base a largo plazo, ya que los componentes de baja calidad se degradan más rápido con el uso continuado.

## VRM (Voltage Regulator Module)

El VRM es el conjunto de componentes electrónicos —mosfets, bobinas (choke) y condensadores— que transforma el voltaje de 12V de la fuente de alimentación a los niveles más bajos que necesita la CPU (típicamente entre 0.8V y 1.5V). Este proceso de conversión debe ser extremadamente preciso: fluctuaciones de apenas unos pocos milivoltios pueden causar inestabilidad en el procesador.

La calidad del VRM se evalúa por varios factores:

**Número de fases:** cada fase del VRM entrega una porción del voltaje total. Un VRM de 12 fases es capaz de distribuir la carga entre más componentes, lo que reduce el calor por fase y mejora la eficiencia. Los procesadores de alto consumo (125W o más) requieren VRM con un número elevado de fases para mantener la estabilidad.

**Tipo de mosfets:** los mosfets *DrMOS* o *Smart Power Stage* (SPS) integran el controlador y el mosfet en un solo paquete, reduciendo la pérdida de energía y mejorando la eficiencia. Los mosfets discretos (separados) son más baratos pero menos eficientes.

**Calidad de los condensadores:** los condensadores de polímero sólido tienen mayor vida útil y mejor estabilidad térmica que los condensadores electrolíticos tradicionales. Las placas base de gama alta utilizan exclusivamente condensadores sólidos.

**Calidad de las bobinas (choke):** las bobinas de ferrita reducen el ruido eléctrico y mejoran la eficiencia de la conversión de voltaje. Las bobinas de hierro pulido (*powdered iron*) son las de mayor calidad.

> Un VRM deficiente no solo causa inestabilidad: también puede acortar la vida útil de la CPU al proporcionar un voltaje ruidoso o inestable. Invertir en una placa base con un VRM de calidad es especialmente importante para sistemas que operarán 24/7 o que utilizan procesadores de alto consumo.

## Importancia de la estabilidad

La estabilidad eléctrica afecta directamente la fiabilidad del sistema. Un voltaje inestable puede causar:

- **Bloqueos aleatorios:** el procesador recibe un voltaje insuficiente durante una operación crítica y el sistema se congela.
- **Errores de memoria:** las fluctuaciones de voltaje afectan a la integridad de los datos almacenados en la RAM, provocando errores de bit.
- **Degradación gradual:** el uso continuado con voltajes inestables acelera el envejecimiento de los componentes electrónicos, reduciendo su vida útil.
- **Daños permanentes:** en el peor caso, un voltaje excesivo puede quemar el procesador o la memoria.

La estabilidad eléctrica es especialmente crítica en entornos servidores, donde los sistemas operan 24 horas al día, 7 días a la semana. Una placa base de servidor incorpora componentes de mayor calidad y circuitos de protección adicionales para garantizar la disponibilidad continua.

## Reloj del sistema (System Clock)

El reloj del sistema es un circuito oscilador que genera una señal de referencia utilizada para sincronizar las operaciones del hardware. La frecuencia del reloj del sistema —típicamente 100 MHz o 25 MHz— es la base a partir de la cual se calculan todas las frecuencias de operación del sistema.

El multiplicador de la CPU toma esta frecuencia de referencia y la multiplica para obtener la frecuencia final del procesador. Por ejemplo, un reloj base de 100 MHz con un multiplicador de 50x produce una frecuencia de 5000 MHz (5 GHz).

La calidad del oscilador del reloj afecta directamente la precisión de las frecuencias de operación. Los osciladores de baja calidad generan *jitter* (variaciones de frecuencia), que pueden causar inestabilidad, especialmente durante el overclock. Las placas base de gama alta incorporan osciladores de alta precisión y circuitos de filtrado para minimizar el jitter.

