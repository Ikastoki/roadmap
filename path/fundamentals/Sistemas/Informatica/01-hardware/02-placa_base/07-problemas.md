# Diagnóstico y problemas comunes

Diagnosticar problemas en una placa base es una habilidad esencial para cualquier técnico o administrador de sistemas. Los fallos pueden manifestarse de múltiples formas: desde un sistema que no enciende hasta bloqueos intermitentes que dificultan la identificación de la causa. Comprender los códigos de diagnóstico, las señales físicas y las herramientas disponibles permite identificar y resolver problemas de forma eficiente.

Un enfoque sistemático de diagnóstico comienza por los síntomas más evidentes y progresa hacia las causas más complejas. Verificar las conexiones básicas antes de sospechar de componentes defectuosos ahorra tiempo y evita reemplazos innecesarios.

## Código POST

El POST (*Power-On Self-Test*) es la primera acción que realiza el firmware al encender el sistema. Si el POST detecta un error crítico, el sistema notifica al usuario de diversas formas:

**Pitidos (beep codes):** los sistemas clásicos emiten secuencias de pitidos a través del altavoz del chasis. Cada secuencia indica un error específico: un pitido largo puede indicar un error de memoria, dos pitidos cortos un error de video, etc. La interpretación de los pitidos varía según el fabricante del firmware (AMI, Phoenix, Award).

**LED de debug:** las placas base modernas incluyen un display LED de dos dígitos que muestra códigos hexadecimales durante el POST. Estos códigos permiten identificar con precisión la fase del arranque donde se produce el fallo. La interpretación de los códigos se encuentra en el manual de la placa base.

**LED de estado:** algunas placas incluyen LEDs dedicados a componentes específicos (CPU, DRAM, VGA, BOOT). Si uno de estos LEDs permanece encendido durante el POST, indica que el fallo está en ese componente.

> Los códigos POST son la herramienta de diagnóstico más valiosa cuando el sistema no completa el arranque. Conocer los códigos de tu placa base antes de tener un problema facilita enormemente la resolución.

## Fallos de arranque

Un fallo de arranque puede deberse a múltiples causas:

**El sistema no enciende:** verificar que la fuente de alimentación está conectada y encendida, que el interruptor de la fuente está en la posición ON, que el botón de encendido de la placa base funciona y que los cables de alimentación (ATX 24 pines, EPS 8 pines) están correctamente conectados.

**El sistema enciende pero no hay imagen:** la CPU puede no estar instalada correctamente, la memoria puede no estar encajada en los slots correspondientes, o la tarjeta gráfica puede no estar conectada. También puede ser un problema de compatibilidad entre CPU y placa base.

**El sistema se reinicia continuamente:** un reinicio cíclico puede indicar un error de memoria, un voltaje inadecuado o un problema con la refrigeración de la CPU. En algunos casos, un BIOS corrupto puede causar este comportamiento.

## Problemas de alimentación

Los problemas de alimentación son algunos de los más comunes y potencialmente dañinos:

- **Voltaje insuficiente:** causado por un VRM defectuoso, cables de alimentación sueltos o una fuente de alimentación inadecuada.
- **Voltaje excesivo:** puede ocurrir durante intentos de overclock mal configurados o por un fallo del regulador de voltaje.
- **Ruido eléctrico:** interferencias electromagnéticas que afectan a la estabilidad del sistema, especialmente visible en bloqueos aleatorios o errores de memoria.

## Problemas de memoria RAM

La memoria es una de las causas más frecuentes de inestabilidad:

- **Módulo defectuoso:** un módulo de RAM con sectores dañados puede causar errores aleatorios, bloqueos o fallos en la carga del sistema operativo. Las herramientas de diagnóstico de memoria (memtest86) permiten verificar la integridad de los módulos.
- **Incompatibilidad:** no todos los módulos de RAM son compatibles entre sí. Mezclar módulos de diferentes fabricantes o velocidades puede causar inestabilidad, especialmente a frecuencias altas.
- **Configuración incorrecta:** habilitar perfiles XMP/EXPO sin verificar la compatibilidad puede provocar fallos. Si el sistema no es estable con XMP, es recomendable intentar la frecuencia base antes de ajustar manualmente.

## Problemas de BIOS/UEFI

El firmware puede experimentar problemas que impiden el funcionamiento normal del sistema:

- **BIOS corrupto:** una actualización interrumpida o un ajuste incorrecto puede corromper el firmware. Las herramientas de recuperación como BIOS Flashback permiten restaurar el firmware sin necesidad de un sistema funcional.
- **Configuración incorrecta:** opciones como el modo de SATA, la secuencia de arranque o las opciones de virtualización mal configuradas pueden impedir la carga del sistema operativo.
- **Firmware desactualizado:** versiones antiguas del firmware pueden tener bugs que afectan a la compatibilidad con nuevos procesadores o dispositivos.

## Sobrecalentamiento

El sobrecalentamiento es un problema común que puede causar throttling (reducción automática de frecuencia), bloqueos o daños permanentes:

- **Pasta térmica seca o mal aplicada:** la interfaz térmica entre el disipador y la CPU debe estar correctamente aplicada para garantizar una transferencia de calor eficiente.
- **Ventilador defectuoso:** un ventilador que no gira o gira a velocidad reducida reduce la capacidad de refrigeración.
- **Flujo de aire deficiente:** la configuración de ventiladores del chasis puede crear zonas de calor que afectan a componentes críticos.

## Señales físicas y visuales

Las placas base incluyen indicadores visuales que pueden ayudar al diagnóstico:

- **LEDs de estado:** indican errores en componentes específicos durante el POST.
- **Condensadores hinchados o rotos:** una señal visual de fallo eléctrico que requiere reemplazo de la placa.
- **Olor a quemado:** indica un componente electrónico que se ha dañado por sobrecalentamiento o cortocircuito.
- **Display LED de códigos:** muestra códigos de diagnóstico durante el arranque.

## Compatibilidad

Los problemas de compatibilidad son frecuentes al mezclar componentes de diferentes generaciones o fabricantes:

- **CPU y chipset:** un procesador puede ser compatible físicamente con el socket pero no con el chipset de la placa base.
- **Memoria y placa base:** no todas las frecuencias de memoria son soportadas por todas las placas base, incluso si el tipo de DDR es compatible.
- **GPU y fuente de alimentación:** las tarjetas gráficas de alta gama requieren conectores de alimentación específicos y una fuente con potencia suficiente.

## Herramientas de diagnóstico

Varias herramientas facilitan el diagnóstico de problemas:

- **memtest86:** herramienta de diagnóstico de memoria que verifica la integridad de los módulos RAM ejecutando patrones de prueba.
- **HwMonitor / HWiNFO:** software de monitorización que muestra temperaturas, voltajes y velocidades de ventiladores en tiempo real.
- **LatencyMon:** diagnostica problemas de latencia del sistema que pueden afectar al rendimiento en tiempo real.
- **CrystalDiskInfo:** verifica el estado de salud de los discos duros y SSD mediante los datos S.M.A.R.T.
- **Multímetro digital:** herramienta física para medir voltajes y continuidad en componentes electrónicos.

