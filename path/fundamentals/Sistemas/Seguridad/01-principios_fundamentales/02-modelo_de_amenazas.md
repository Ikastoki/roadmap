# Modelo de Amenazas

Antes de implementar cualquier measure de seguridad es necesario responder tres preguntas fundamentales: que estamos protegiendo, contra que nos estamos defendiendo, y que tan probable es que algo salga mal. Un modelo de amenazas estructura estas preguntas en un marco analitico que permite tomar decisiones informadas sobre donde invertir recursos de seguridad. Sin este proceso, la seguridad se convierte en una coleccion reactiva de herramientas y configuraciones que protegen aleatoriamente mientras dejan areas criticas desprotegidas.

## Identificacion de activos

El primer paso es determinar que activos posee la organizacion y cual es su valor. Los activos no son exclusivamente datos: incluyen hardware, software, servicios, reputacion, tiempo de personal y capacidad operativa. Un servidor web es un activo, pero tambien lo son las credenciales de acceso a el, los certificados SSL que posee, el trafico de red que genera y la disponibilidad del servicio que ofrece.

La clasificacion de activos permite asignar niveles de criticidad. Un servidor que almacena datos financieros tiene un valor intrinsico mayor que un servidor de pruebas interno. Sin embargo, esta clasificacion debe considerar dependencias: un servidor de logs que parece poco importante puede ser critico si es necesario para detectar intrusiones, o si contiene registros de auditoria requeridos por regulacion. El error comun es clasificar solo los activos visibles e ignorar los activos derivados como metadatos, registros de acceso y configuraciones que permiten operar sobre los activos primarios.

## Amenazas

Una amenaza es cualquier entidad o evento que tiene el potencial de causar dano a un activo. Las amenazas pueden clasificarse en diversas categorias: amenazas internas (empleados descontentos, errores humanos), amenazas externas (hackers organizados, grupos state-sponsored), amenazas ambientales (desastres naturales, fallos de energia) y amenazas estructurales (obsolescencia tecnica, deuda tecnica acumulada).

Es importante distinguir entre amenazas y vectores de ataque. Un hacker es una amenaza; la explotacion de una vulnerabilidad en el servidor de correo es un vector de ataque. El mismo actor amenazante puede utilizar multiples vectores, y el mismo vector puede ser explotado por diferentes amenazas. Esta distincion permite analizar cada vector individualmente y determinar contra cuales amenazas es efectiva cada contramedida.

La amenaza mas peligrosa no es necesariamente la mas sofisticada, sino la mas probable. Un ransomware que utiliza credenciales por defecto en un servicio expuesto a internet es una amenaza mucho mas relevante que un ataque de estado-nacion altamente complejo, simplemente por su frecuencia de ocurrencia.

## Vulnerabilidades

Las vulnerabilidades son debilidades que una amenaza puede explotar para causar dano. A diferencia de las amenazas, que son externas al sistema, las vulnerabilidades residen dentro de la infraestructura: un software desactualizado, una configuracion por defecto, una arquitectura deficiente o una falta de monitoreo. Las vulnerabilidades no son inherentemente peligrosas; solo se convierten en un problema cuando una amenaza las identifica y las explota.

Las vulnerabilidades pueden clasificarse por su naturaleza. Las vulnerabilidades de software (CVEs publicadas) son conocidas y generalmente tienen parches disponibles. Las vulnerabilidades de configuracion son errores humanos como servicios expuestos innecesariamente o credenciales por defecto. Las vulnerabilidades de diseno son problemas arquitectonicos que no se pueden resolver con un parche, como la falta de segmentacion de red o la ausencia de principio de minimo privilegio en una base de datos.

El registro CVE (Common Vulnerabilities and Exposures) cataloga vulnerabilidades conocidas con un identificador unico y una puntuacion CVSS (Common Vulnerability Scoring System) que mide su severidad. Sin embargo, la existencia de un CVE no implica que una organizacion sea vulnerable: si el software afectado no se utiliza, la vulnerabilidad es irrelevante. La gestion efectiva de vulnerabilidades requiere primero inventariar los activos, luego cruzar ese inventario con las vulnerabilidades conocidas, y finalmente evaluar la exposicion real.

## Riesgo

El riesgo es la interseccion entre amenazas, vulnerabilidades y activos. Se expresa comunmente mediante la ecuacion:

**Riesgo = Probabilidad × Impacto**

La probabilidad determina que tan probable es que una amenaza explote una vulnerabilidad especifica. Depende de factores como la exposicion del activo (un servidor publico tiene mayor probabilidad de ser atacado que uno interno), la sofisticacion de las amenazas activas en el sector y la efectividad de las contramedidas existentes. La probabilidad nunca es cero: siempre existe la posibilidad de que un ataque tenga exito.

El impacto mide las consecuencias si el ataque se materializa. Incluye perdidas financieras directas, danos a la reputacion, sanciones regulatorias, perdida de propiedad intelectual y discontinuidad operativa. El impacto debe evaluarse en terminos cuantificables cuando sea posible: horas de downtime multiplicadas por el costo por hora, registros comprometidos multiplicados por el costo promedio por registro perdido.

La combinacion de probabilidad e impacto produce un mapa de riesgos que permite priorizar. Un riesgo con alta probabilidad y alto impacto requiere atencion inmediata. Un riesgo con baja probabilidad pero alto impacto (como un desastre natural que destruye un data center) justifica la inversion en planes de continuidad. Un riesgo con alta probabilidad pero bajo impacto puede mitigarse con controles operativos simples. Un riesgo con baja probabilidad y bajo impacto puede aceptarse activamente, documentando la decision.

## Gestion del riesgo

Una vez calculado el riesgo, existen cuatro estrategias fundamentales para tratarlo. La mitigacion implementa controles que reducen la probabilidad o el impacto: parchar un servidor reduce la probabilidad de explotacion, mientras que tener un backup reduce el impacto de un ransomware. La transferencia desplaza el riesgo a otra entidad, como cuando se contrata un seguro cibernetico o se Externaliza un servicio a un proveedor con mejores capacidades de seguridad. La aceptacion reconoce que el costo de mitigar el riesgo supera el beneficio esperado, y decide convivir con el residual. La evitacion elimina la actividad que genera el riesgo, como no desplegar una funcionalidad particular que introduce una vulnerabilidad critica.

La clave es que ninguna organizacion puede eliminar todos los riesgos. El objetivo es mantener el riesgo residual en un nivel aceptable para la organizacion, entendiendo que ese nivel es dinamico y debe reevaluarse periodicamente. Las nuevas amenazas, los cambios en la infraestructura y la evolucion del negocio modifican constantemente el perfil de riesgo, haciendo del modelo de amenazas un documento vivo y no un ejercicio puntual.

---

