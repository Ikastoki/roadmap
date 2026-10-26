# Rieles de voltaje

Los rieles de voltaje son los canales independientes a través de los cuales la fuente de alimentación distribuye energía a los diferentes componentes del sistema. Cada riel proporciona un voltaje específico y tiene una capacidad máxima de corriente que limita cuántos componentes pueden alimentarse simultáneamente desde ese riel. La gestión adecuada de los rieles es esencial para garantizar la estabilidad del sistema, especialmente en configuraciones con componentes de alto consumo.

## Rieles de voltaje

Una fuente de alimentación ATX estándar proporciona tres voltajes principales:

- **+12V:** el riel más importante, que alimenta la CPU, la GPU, los ventiladores y la mayoría de los componentes mecánicos.
- **+5V:** alimenta dispositivos de almacenamiento (discos duros, SSD SATA), puertos USB y la lógica de la placa base.
- **+3.3V:** alimenta la memoria RAM, el chipset de la placa base y algunos dispositivos de baja potencia.

Además de estos tres rieles principales, las fuentes también proporcionan voltajes auxiliares como -12V y +5VSB (*standby*), que mantiene ciertos componentes energizados incluso cuando el sistema está apagado (para funciones como Wake-on-LAN o encendido por teclado).

## El riel de +12V

El riel de +12V es con diferencia el más crítico de la fuente de alimentación. Los componentes que más energía consumen —CPU y GPU— dependen exclusivamente de este riel, y su capacidad determina directamente la potencia máxima utilizable del sistema.

En las fuentes modernas de gama alta, el riel de +12V puede proporcionar más de 50A de corriente, equivalente a más de 600W de potencia. Una GPU de gama alta como la NVIDIA RTX 4090 puede consumir hasta 450W bajo carga, lo que requiere un riel de +12V capaz de mantener un voltaje estable incluso con fluctuaciones rápidas de demanda.

La estabilidad del riel de +12V es especialmente importante para la GPU, que puede pasar de estar inactiva (consuming apenas unos pocos vatios) a plena carga en milisegundos. Una fuente con un riel de +12V deficiente puede no ser capaz de responder a estos cambios repentinos de demanda, causando caídas de voltaje que provocan bloqueos o reinicios.

## +5V

El riel de +5V alimenta dispositivos que requieren menor potencia pero que son críticos para el funcionamiento del sistema:

- **Dispositivos de almacenamiento:** los discos duros y los SSD SATA obtienen su energía principal del riel de +5V.
- **Puertos USB:** cada puerto USB proporciona hasta 500mA a 5V (2.5W) en USB 2.0 y hasta 900mA (4.5W) en USB 3.0.
- **Lógica de la placa base:** ciertos circuitos de la placa base operan a 5V.
- **Iluminación RGB:** los strips de LEDs y otros dispositivos de iluminación suelen requerir +5V.

## +3.3V

El riel de +3.3V alimenta componentes de baja potencia que operan a voltajes reducidos:

- **Memoria RAM:** los módulos DDR operan a voltajes reducidos (1.2V para DDR4, 1.1V para DDR5), pero obtienen su energía a través de reguladores que convierten el +3.3V al voltaje requerido.
- **Chipset:** el chipset de la placa base y otros circuitos integrados operan a 3.3V.
- **Algunos dispositivos M.2:** los SSD M.2 pueden obtener parte de su energía del riel de 3.3V.

## La importancia del riel único

En las fuentes modernas de gama alta, es común encontrar diseños con riel único de +12V, donde toda la capacidad de corriente está disponible en un solo riel. Esto simplifica la distribución de energía y elimina el problema de la asignación de carga entre múltiples rieles.

En fuentes con múltiples rieles de +12V, la capacidad de corriente se divide entre ellos, lo que puede causar problemas si un componente consume más de la capacidad de un solo riel. Por ejemplo, si un riel de +12V tiene una capacidad de 30A y la GPU requiere 40A bajo carga, el sistema puede bloquearse aunque la fuente tenga capacidad total suficiente.

> Las fuentes con riel único de +12V son preferibles para configuraciones con GPU de alto consumo, ya que garantizan que toda la capacidad de corriente está disponible para el componente que más la necesita. Verificar si una fuente tiene riel único o múltiples rieles es un paso frecuentemente olvidado al elegir una fuente de alimentación.

