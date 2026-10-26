# Memoria de solo lectura (ROM)

La memoria de solo lectura (*Read-Only Memory*) es una categoría de memoria no volátil diseñada para almacenar datos que deben permanecer disponibles sin energía y que generalmente no se modifican durante el funcionamiento normal del sistema. A diferencia de la RAM, la ROM conserva su contenido al apagar el equipo, lo que la hace ideal para firmware, microcódigos y datos de configuración que el sistema necesita en el momento del arranque.

Las tecnologías de ROM han evolucionado significativamente desde los primeros circuitos programables de la década de 1970. Lo que comenzó como chips con contenido grabado de fábrica se transformó en una familia diversa de tecnologías que combinan la permanencia de la no volatilidad con distintos grados de reprogramabilidad.

## Tipos de ROM

**PROM (*Programmable Read-Only Memory*):** fue la primera tecnología de ROM programable por el usuario. Cada chip PROM contiene un arreglo de fusibles que pueden ser quemados una sola vez mediante un dispositivo programador especial. Una vez quemados los fusibles, el contenido queda grabado permanentemente y no puede modificarse. La PROM permitía a los fabricantes personalizar el firmware sin necesidad de diseñar un chip personalizado, pero su naturaleza de una sola escritura la hacía inflexible y costosa para desarrollo.

**EPROM (*Erasable Programmable Read-Only Memory*):** introducida en 1971, representó un avance significativo al permitir el borrado y reprogramación del contenido. La EPROM se borra exponiendo el chip a luz ultravioleta a través de una ventana de cuarzo visible en la parte superior del encapsulado. El proceso completo de borrado requiere entre 15 y 20 minutos de exposición continua. Aunque era reprogramable, la necesidad de un equipo especializado para el borrado limitaba su uso a entornos de desarrollo y fabricación.

**EEPROM (*Electrically Erasable Programmable Read-Only Memory*):** evolucionó el concepto de EPROM al permitir el borrado eléctrico, sin necesidad de luz ultravioleta. La EEPROM puede ser borrada y reprogramada byte a byte directamente en el circuito, lo que la hizo mucho más práctica para aplicaciones donde era necesario actualizar el firmware periódicamente. Sin embargo, la EEPROM es más lenta que la EPROM en escritura y tiene un número limitado de ciclos de borrado (típicamente entre 100,000 y 1,000,000).

**Flash Memory:** es la evolución moderna de la EEPROM y la tecnología que domina actualmente tanto el almacenamiento como las memorias no volátiles embebidas. A diferencia de la EEPROM, la Flash se borra en bloques grandes en lugar de byte a byte, lo que permite una mayor densidad y un coste significativamente menor. La Flash se divide en dos variantes principales:

- **NOR Flash:** permite acceso aleatorio byte a byte, similar a la RAM. Es ideal para ejecución de código (*execute-in-place*) y se utiliza en sistemas embebidos donde el procesador necesita ejecutar directamente el código desde la memoria Flash sin copiarlo a RAM previamente.

- **NAND Flash:** ofrece mayor densidad y menor coste por bit, pero solo permite acceso por bloques. Es la tecnología base de los SSD, unidades USB y tarjetas de memoria. No permite ejecución directa de código (*execute-in-place*), por lo que el sistema operativo debe cargar los datos en RAM antes de ejecutarlos.

> La distinción entre NOR y NAND Flash es fundamental: NOR preserva la capacidad de acceso aleatorio característica de la ROM tradicional, mientras que NAND sacrifica esta capacidad a cambio de densidad y coste significativamente mejores.

## Su uso en el firmware (BIOS/UEFI)

El uso más visible de la memoria Flash en un PC de escritorio es el chip que almacena el firmware de la placa base (BIOS o UEFI). Este chip Flash contiene las instrucciones iniciales que el procesador ejecuta al encender el sistema, antes de que cualquier otro componente esté operativo.

El firmware de la placa base realiza una serie de funciones críticas durante el arranque:

- **Inicialización del hardware:** configura el procesador, la memoria RAM, los controladores de E/S y los dispositivos de almacenamiento a sus estados operativos.
- **Prueba POST (*Power-On Self-Test*):** verifica que los componentes críticos funcionan correctamente antes de entregar el control al sistema operativo.
- **Carga del gestor de arranque:** busca y ejecuta el bootloader del sistema operativo desde el dispositivo de almacenamiento configurado.
- **Configuración del sistema:** almacena la configuración del usuario (orden de arranque, configuración de overclock, permisos de hardware) en un chip CMOS alimentado por una pila auxiliar.

La naturaleza Flash del chip permite las actualizaciones de firmware (*flash BIOS*) que corrigen errores, añaden compatibilidad con nuevos procesadores o implementan nuevas funciones de seguridad. Este proceso de actualización, aunque cada vez más seguro gracias a mecanismos como BIOS Flashback, siempre conlleva un riesgo: una interrupción durante la escritura puede corromper el firmware y dejar la placa base inoperativa.

> La implementación de BIOS Flashback por parte de los fabricantes de placas base permite actualizar el firmware sin necesidad de procesador, memoria o tarjeta gráfica instalados. Se utiliza una memoria USB con el archivo de firmware y un botón dedicado en la placa, lo que permite recuperar una placa base con firmware corrupto que de otro modo sería inservible.

