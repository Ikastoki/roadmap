# Fabricación y litografía

La fabricación de un procesador es uno de los procesos industriales más complejos jamás desarrollados por la humanidad. Un chip moderno contiene miles de millones de transistores —cada uno menor que un virus— dispuestos con una precisión atómica sobre una oblea de silicio del tamaño de un plato. Comprender cómo se fabrican estos dispositivos es entender la frontera de la ingeniería moderna y las razones por las que la fabricación de chips está concentrada en apenas unas pocas empresas en el mundo.

La litografía, el proceso clave de la fabricación de semiconductores, utiliza luz ultravioleta extrema para grabar patrones microscópicos en el silicio. Cada nueva generación de litografía ha permitido reducir el tamaño de los transistores, aumentando su número y eficiencia en cada chip.

## El proceso de fabricación

La fabricación de un procesador comienza con arena de cuarzo, que se purifica hasta obtener silicio policristalino de pureza extrema (99.9999999%). Este silicio se funde y crece en cilindros monocristalinos (*ingots*) que se cortan en obleas (*wafers*) circulares de hasta 300 mm de diámetro.

Sobre estas obleas se realizan cientos de pasos de fabricación que incluyen:

**Deposición:** se aplican capas extremadamente delgadas de materiales conductores, aislantes o semiconductoras sobre la superficie de la oblea.

**Litografía:** se utiliza luz ultravioleta a través de una máscara (*mask* o *reticle*) para transferir patrones a la capa sensibilizada (*photoresist*) sobre la oblea. Los patrones definen la forma y ubicación de los transistores, interconexiones y otros elementos.

**Grabado (*Etching*):** se eliminan las zonas no protegidas por el photoresist, tallando los patrones en la capa subyacente. El grabado puede ser químico (húmedo) o físico (plasma).

**Implantación iónica:** se inyectan átomos dopantes (boro, fósforo, arsénico) en zonas específicas del silicio para modificar sus propiedades eléctricas y crear las regiones P y N de los transistores.

**Pulido químico mecánico (CMP):** se iguala la superficie de la oblea después de cada capa para garantizar que los siguientes pasos de litografía tengan la precisión necesaria.

Este proceso se repite cientos de veces, construyendo capa sobre capa hasta completar los circuitos integrados. Una oblea puede contener cientos de chips (*dies*) individuales, que se separan, prueban y empaquetan para su venta.

> El tiempo medio de fabricación de un chip avanzado es de aproximadamente 3 meses desde el inicio del proceso hasta la oblea terminada. Cada oblea puede pasar por más de 1.000 pasos de fabricación.

## Micras y nanómetros

El tamaño de los transistores se mide en nanómetros (nm), una unidad equivalente a la milmillonésima parte de un metro. Los nombres de los nodos de fabricación —7nm, 5nm, 3nm— representan la generación tecnológica del proceso, aunque no corresponden directamente a una dimensión física del transistor.

La evolución del tamaño de los transistores ha sido espectacular:

- **1971 (Intel 4004):** 10.000 nm (10 micras)
- **1995 (Pentium Pro):** 350 nm
- **2001 (Pentium 4):** 130 nm
- **2007 (Core 2):** 45 nm
- **2012 (Ivy Bridge):** 22 nm
- **2020 (Apple M1):** 5 nm
- **2023 (Apple M3):** 3 nm

Cada reducción de nodo permite:

- **Mayor densidad de transistores:** más transistores por mm², lo que habilita funcionalidades más complejas o más núcleos en el mismo chip.
- **Menor consumo energético:** los transistores más pequeños requieren menos energía para conmutar.
- **Mayor velocidad:** las distancias de propagación de la señal se reducen, permitiendo frecuencias más altas.

> El término "nm" en los nodos modernos es más un indicador de generación que una medida física real. El "3nm" de TSMC no significa que los transistores tengan 3 nanómetros de largo; es una denominación comercial que indica la evolución respecto al nodo anterior.

## La Ley de Moore

Formulada por Gordon Moore en 1965, la Ley de Moore predijo que el número de transistores en un circuito integrado se duplicaría aproximadamente cada dos años. Esta predicción se cumplió con asombrosa precisión durante casi medio siglo, impulsando una aceleración exponencial en el poder de cálculo y una reducción continua en el coste por transistor.

La Ley de Moore no es una ley física, sino una observación empírica que se ha convertido en una profecía autocumplida. Los fabricantes de semiconductores han invertido miles de millones de dólares en I+D para mantener el ritmo de duplicación, creando un ciclo virtuoso de innovación.

Sin embargo, la Ley de Moore está alcanzando sus límites físicos. Los transistores de los nodos más avanzados miden apenas unos pocos átomos de ancho, y las reducciones adicionales enfrentan barreras cuánticas como el tunelamiento electrónico, donde los electrones atraviesan barreras que deberían ser impenetrables a esas escalas.

Las soluciones a estas limitaciones incluyen:

- **Nuevas estructuras de transistores:** los transistores GAA (*Gate-All-Around*) o *nanosheet* reemplazan el diseño FinFET, rodeando el canal por completo para mejorar el control.
- **Apilamiento 3D:** *chiplet* y *3D stacking* permiten apilar capas de circuitos verticalmente, aumentando la densidad sin reducir el tamaño del transistor.
- **Nuevos materiales:** el grafeno, los semiconductores orgánicos y otros materiales exóticos podrían reemplazar al silicio en futuras generaciones.

> La Ley de Moore ha sido el motor de la revolución digital. Sin ella, no existirían los smartphones, los servidores cloud ni la inteligencia artificial tal como la conocemos. Su eventual fin plantea desafíos fundamentales para la industria tecnológica.

