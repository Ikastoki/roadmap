# Protecciones eléctricas

Las protecciones eléctricas son circuitos integrados en la fuente de alimentación que detectan y responden a condiciones anormales que podrían dañar los componentes del sistema. Estas protecciones actúan como un seguro: en condiciones normales permanecen inactivas, pero si se produce una anomalía, interrumpen o limitan el flujo de energía para prevenir daños.

Una fuente sin protecciones adecuadas puede convertirse en un riesgo para todo el sistema. Un cortocircuito en la GPU, por ejemplo, puede inyectar corrientes enormes a través de la fuente, provocando daños en la placa base, la CPU e incluso incendios si la fuente no dispone de protección contra sobrecorriente.

## Sobretensión (OVP)

La protección contra sobretensión (*Over Voltage Protection*) monitoriza los voltajes de salida y los desconecta si superan un umbral predefinido. Si el voltaje del riel de +12V sube por encima de 13.2V (10% por encima del nominal), la OVP corta la energía para evitar daños en los componentes.

La OVP es especialmente crítica para proteger la CPU y la GPU, que son sensibles a fluctuaciones de voltaje. Un fallo en el regulador de voltaje de la fuente puede provocar un pico de tensión que, sin OVP, quemaría instantáneamente el procesador.

## Sobrecarga (OPP)

La protección contra sobrecarga (*Over Power Protection*) limita la potencia total que la fuente puede entregar. Si la demanda de energía supera la capacidad máxima de la fuente, la OPP reduce la potencia o apaga la fuente para evitar el sobrecalentamiento de sus componentes internos.

La OPP es importante en configuraciones donde la GPU puede demandar picos de energía superiores a su TDP nominal. Sin esta protección, la fuente podría operar fuera de sus especificaciones durante períodos prolongados, acelerando su degradación.

## Cortocircuito (SCP)

La protección contra cortocircuitos (*Short Circuit Protection*) detecta corrientes anormalmente altas causadas por un cortocircuito y corta la energía instantáneamente. Un cortocircuito puede ocurrir por un cable dañado, un componente defectuoso o un error de montaje.

La SCP es la protección más rápida: debe actuar en microsegundos para prevenir daños catastróficos. Una fuente de calidad SCP en menos de 100 microsegundos, limitando la corriente de pico a niveles seguros.

## Subtensión (UVP)

La protección contra subtensión (*Under Voltage Protection*) monitoriza los voltajes de salida y los desconecta si caen por debajo de un umbral predefinido. Un voltaje insuficiente puede causar comportamientos erráticos en los componentes: bloqueos, reinicios o corrupción de datos.

La UVP es especialmente relevante para el riel de +12V, que alimenta la CPU y la GPU. Una caída de voltaje bajo carga puede provocar que la GPU deje de funcionar correctamente, causando artefactos gráficos o bloqueos del sistema.

## Protección contra sobrecalentamiento (OTP)

La protección contra sobrecalentamiento (*Over Temperature Protection*) monitoriza la temperatura interna de la fuente y la apaga si supera un límite seguro. El sobrecalentamiento puede ocurrir por ventilación insuficiente, polvo acumulado o funcionamiento a plena carga durante períodos prolongados.

> Las protecciones eléctricas no son opcionales: una fuente de alimentación sin OVP, SCP y OPP está comprometiendo la seguridad de todo el sistema. Al elegir una fuente, verificar que incluye todas las protecciones básicas es tan importante como verificar su potencia o su eficiencia.

## La importancia de un buen diseño de protección

La calidad de las protecciones no se mide solo por su existencia, sino por la velocidad y precisión con que actúan. Una fuente barata puede incluir protecciones en el papel pero con tiempos de respuesta lentos o umbrales poco precisos que no protegen adecuadamente los componentes.

Los fabricantes de renombre (Seasonic, Corsair, be quiet!, Super Flower) invierten en circuitos de protección robustos con tiempos de respuesta ultrarrápidos. Los fabricantes genéricos pueden incluir protecciones básicas que funcionan, pero con margenes de seguridad más estrechos y tiempos de respuesta más lentos.

