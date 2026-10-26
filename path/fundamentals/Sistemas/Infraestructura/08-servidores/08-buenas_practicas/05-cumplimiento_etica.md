# Cumplimiento y Etica

La infraestructura tecnica no existe en un vacio legal. Cada servidor que almacena datos personales, cada base de datos que procesa informacion de clientes y cada servicio que transmite datos esta sujeto a un marco normativo que establece obligaciones especificas para quien los opera. El cumplimiento normativo no es una opcion ni un adorno corporativo: es un requisito legal cuyo incumplimiento conlleva consecuencias economicas, reputacionales e incluso penales.

## Marco normativo

### Reglamento General de Proteccion de Datos (RGPD)

El RGPD, aplicable en toda la Union Europea desde mayo de 2018, establece las reglas fundamentales para el tratamiento de datos personales. Desde la perspectiva de la infraestructura, el RGPD impone requisitos concretos: los datos personales deben cifrarse en reposo y en transito, los accesos a datos deben registrarse, los usuarios deben poder solicitar la eliminacion de sus datos y la organizacion debe notificar brechas de seguridad dentro de las 72 horas siguientes a su deteccion.

El principio de minimizacion de datos exige que la infraestructura solo almacene y procese los datos estrictamente necesarios para el proposito informado al usuario. Un servidor de logs no debe contener direcciones IP completas si no son necesarias. Una base de datos de usuarios no debe mantener contrasenas antiguas. Cada dato almacenado sin necesidad es un riesgo adicional.

### Ley de Servicios de la Sociedad de la Informacion (LSSI)

En Espana, la LSSI complementa al RGPD estableciendo requisitos especificos sobre cookies, comunicaciones electronicas y servicios de la sociedad de la informacion. Desde la infraestructura, esto se traduce en la necesidad de configurar servidores web que gestionen correctamente el consentimiento de cookies, que mantengan registros de comunicaciones y que proporcionen mecanismos de baja efectivos.

### PCI DSS

Para organizaciones que procesan datos de tarjetas de pago, el estandar PCI DSS establece requisitos tecnicos especificos: segmentacion de red, cifrado de datos de tarjetas, monitoreo de accesos y pruebas periodicas de seguridad. Estos requisitos afectan directamente al diseno de la infraestructura: los servidores que manejan datos de pago deben estar en segmentos de red aislados, con politicas de firewall especificas y con monitoreo intensivo.

## Notificacion de brechas de seguridad

El RGPD obliga a notificar a la autoridad de control competente las brechas de seguridad que afecten a datos personales en un plazo de 72 horas desde su deteccion. Si la brecha supone un alto riesgo para los derechos y libertades de las personas afectadas, tambien se debe notificar a estos directamente.

La notificacion debe incluir la naturaleza de la brecha, las categorias y numero aproximado de personas afectadas, las posibles consecuencias de la brecha y las medidas adoptadas o propuestas para abordarla. Este proceso requiere que la organizacion disponga de procedimientos claros de deteccion, clasificacion y comunicacion de incidentes.

### Preparacion operativa

La preparacion para notificar brechas no es algo que se improvisa en el momento del incidente. Requiere procedimientos documentados que definan quien detecta la brecha, quien la clasifica, quien toma la decision de notificar, que informacion se incluye en la notificacion y como se comunica a los afectados. Estos procedimientos deben probarse periodicamente mediante simulacros.

Un error comun es retrasar la deteccion por falta de monitoreo o subestimar la gravedad de una brecha por incomrension de sus implicaciones legales. La infraestructura debe implementar mecanismos de deteccion que identifiquen accesos no autorizados, exfiltraciones de datos y configuraciones de seguridad incorrectas, y generar alertas que se evaluen en un plazo determinado.

## Etica en la gestion de datos

Mas alla del cumplimiento estricto de la normativa, la gestion de infraestructura implica responsabilidades eticas que trascienden lo legal.

### Transparencia en el tratamiento de datos

Los usuarios tienen derecho a saber que datos se almacenan sobre ellos, como se utilizan y con quien se comparten. La infraestructura debe facilitar esta transparencia: los sistemas de registro deben ser consultables, los procesos de eliminacion de datos deben ser ejecutables y las politicas de retencion deben ser claras y aplicables.

### Responsabilidad en el diseno

El concepto de "privacy by design" establece que la proteccion de datos debe incorporarse desde el diseno de la infraestructura, no tratarse como una capa posterior. Esto implica cifrar datos por defecto, minimizar la recogida de informacion, anonimizar datos cuando sea posible y implementar controles de acceso robustos desde el primer dia.

### Limitacion del poder tecnologico

La infraestructura otorga a sus administradores un poder significativo sobre los datos de las personas. Este poder conlleva una responsabilidad etica de no abusar de el. Acceder a datos de usuarios sin justificacion, manipular logs para ocultar actividades o utilizar la infraestructura para vigilancia no autorizada son actuaciones que, ademas de ser ilegales, violan la confianza que los usuarios depositan en la organizacion.

## Auditorias y revisiones

Las auditorias periodicas de seguridad y cumplimiento son el mecanismo que verifica que las politicas se aplican efectivamente en la practica. Una politica de contrasenas es inutil si los usuarios siguen usando credenciales debiles. Un procedimiento de acceso es irrelevante si los administradores lo eluden con atajos.

Las auditorias deben ser realizadas por personas o equipos independientes de quienes gestionan la infraestructura, para evitar conflictos de interes. Los resultados de las auditorias deben documentarse, y las no conformidades identificadas deben corregirse en plazos definidos. El incumplimiento reiterado de una politica de seguridad debe tratarse como un riesgo operacional serio, no como una incomveniencia administrativa.

