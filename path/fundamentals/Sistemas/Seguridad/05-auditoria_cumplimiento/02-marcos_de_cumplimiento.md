# Marcos de Cumplimiento

Los marcos de cumplimiento son conjuntos estructurados de normas, directrices y mejores practicas que establecen como una organizacion debe proteger la informacion que gestiona. No son documentos teoricos: son marcos operativos que definen controles especificos, procedimientos de verificacion y mecanismos de mejora continua. Existen porque la seguridad de la informacion no puede dejarse a la improvisacion de cada organizacion; el mercado necesita estandares comunes que permitan comparar, evaluar y confiar en las practicas de seguridad de terceros. Un proveedor cloud que cumple con SOC 2 esta comunicando a sus clientes que ha implementado controles verificables, no simplemente que "se toma la seguridad en serio".

El cumplimiento normativo no es simplemente una lista de verificacion que se completa una vez y se archiva. Es un enfoque de gestion del riesgo que obliga a la organizacion a identificar sus activos criticos, evaluar las amenazas que enfrentan, implementar controles proporcionales y demostrar periodicamente que esos controles siguen siendo efectivos. Esta distincion es critica: una organizacion que "cumple" un marco solo para obtener un certificado esta perdiendo el valor real del proceso. El certificado es una consecuencia, no el objetivo.

## ISO 27001

ISO 27001 es el estandar internacional para la gestion de la seguridad de la informacion (ISMS - Information Security Management System). Define los requisitos para establecer, implementar, mantener y mejorar continuamente un sistema de gestion de seguridad de la informacion dentro del contexto de la organizacion.

El marco se estructura en dos partes principales. La primera establece los requisitos del sistema de gestion (controles organizacionales, Plan-Do-Check-Act), y la segunda lista 93 controles de seguridad organizados en 4 temas:

- **Organizacionales**: politicas de seguridad, asignacion de responsabilidades, gestion de activos, classification de informacion, relaciones con proveedores.
- **Personas**: concienciacion y formacion, procedimientos para contratacion y terminacion de empleo.
- **Fisicos**: control de areas seguras, proteccion contra amenazas fisicas, seguridad en instalaciones.
- **Tecnologicos**: controles de acceso, criptografia, seguridad del desarrollo, gestion de vulnerabilidades, registro y monitorizacion.

La certificacion ISO 27001 implica una auditoria externa realizada por un organismo certificador acreditado. Esta auditoria verifica que el ISMS esta implementado de forma efectiva y que la organizacion cumple con todos los requisitos del estandar. La certificacion no es permanente: se realizan auditorias de seguimiento anuales y una recertificacion cada tres anos.

> ISO 27001 no prescribe controles especificos para cada amenaza. Define un marco de trabajo que obliga a la organizacion a analizar su propio contexto y seleccionar los controles mas adecuados segun su perfil de riesgo. Esto lo hace adaptable a organizaciones de cualquier tamano o sector.

## SOC 2

SOC 2 (System and Organization Controls 2) es un marco de auditoria desarrollado por el AICPA (American Institute of Certified Public Accountants) diseñado especificamente para proveedores de servicios SaaS, cloud y tecnologicos que almacenan o procesan datos de clientes. A diferencia de ISO 27001, que es un estandar internacional certificable, SOC 2 es un estandar de auditoria que resulta en un informe emitido por un auditor externo.

SOC 2 se basa en 5 principios de confianza (Trust Services Criteria):

- **Seguridad**: proteccion de los sistemas contra acceso no autorizado, tanto fisico como logico. Es el unico principio obligatorio para todos los informes SOC 2.
- **Disponibilidad**: los sistemas estan operativos y funcionales segun lo acordado contractualmente.
- **Integridad de procesamiento**: los sistemas procesan datos de forma completa, precisa y oportuna.
- **Confidencialidad**: la informacion clasificada como confidencial se protege segun lo acordado.
- **Privacidad**: la informacion personal se recoge, usa, retiene y divulga de forma coherente con las practicas declaradas.

SOC 2 Tipo I evalua el diseno de los controles en un momento especifico. SOC 2 Tipo II evalua la efectividad operativa de esos controles durante un periodo (tipicamente 6 o 12 meses). Para un SaaS que busca ganar confianza de clientes empresariales, un SOC 2 Tipo II es significativamente mas valioso porque demuestra que los controles no solo existen sino que funcionan en la practica.

## PCI DSS

PCI DSS (Payment Card Industry Data Security Standard) es el estandar de seguridad para organizaciones que procesan, almacenan o transmiten datos de tarjetas de pago. Fue creado por el PCI SSC (Payment Card Industry Security Standards Council), un organismo fundado por las cinco grandes marcas de tarjetas: Visa, Mastercard, American Express, Discover y JCB.

El estandar define 12 requisitos agrrupados en 6 objetivos:

1. Instalar y mantener un firewall para proteger los datos del titular de la tarjeta.
2. No utilizar contrasenas por defecto suministradas por los proveedores.
3. Proteger los datos almacenados del titular de la tarjeta.
4. Cifrar la transmision de datos del titular de la tarjeta sobre redes publicas.
5. Utilizar y actualizar regularmente un software antivirus.
6. Desarrollar y mantener sistemas y aplicaciones seguros.
7. Restringir el acceso a los datos del titular de la tarjeta segun la necesidad del negocio.
8. Asignar un ID unico a cada persona con acceso al sistema.
9. Restringir el acceso fisico a los datos del titular de la tarjeta.
10. Monitorizar y auditar regularmente el acceso a los recursos de la red y a los datos del titular de la tarjeta.
11. Mantener una politica de seguridad de la informacion que aborde la seguridad de los datos.

El nivel de cumplimiento requerido (y el tipo de validacion) depende del volumen de transacciones que procese la organizacion anualmente. Las grandes empresas pueden necesitar una auditoria QSA (Qualified Security Assessor) on-site, mientras que las mas pequenas pueden aut-certificarse mediante un cuestionario SAQ (Self-Assessment Questionnaire).

> PCI DSS es uno de los marcos con mayor rigor de cumplimiento porque las sanciones por incumplimiento pueden incluir multas significativas, la perdida del privilegio de procesar pagos con tarjeta y la responsabilidad legal por fraude resultante.

## HIPAA

HIPAA (Health Insurance Portability and Accountability Act) es una ley estadounidense de 1996 que establece estandares para la proteccion de la informacion de salud personal (PHI - Protected Health Information). Aunque es una ley de EE.UU., su alcance afecta a cualquier organizacion que maneje datos de salud de ciudadanos estadounidenses, incluyendo proveedores de servicios cloud internacionales.

HIPAA define tres conjuntos de reglas principales:

- **Privacy Rule**: establece los limites sobre quién puede acceder a la PHI y bajo que condiciones. Define el principio de "menor acceso necesario" y los derechos de los pacientes sobre sus datos.
- **Security Rule**: especifica las salvaguardas electronicas obligatorias para proteger la ePHI (electronic Protected Health Information). Incluye controles administrativos, fisicos y tecnicos.
- **Breach Notification Rule**: obliga a notificar a los afectados, al Departamento de Salud y en algunos casos a los medios de comunicacion cuando se produce una brecha de seguridad que afecte a mas de 500 personas.

El Security Rule exige medidas como el cifrado de datos en reposo y en transmision, el control de acceso basado en roles, las copias de seguridad periodicas, la monitorizacion de accesos y los planes de respuesta ante desastres. El incumplimiento puede resultar en sanciones que van desde 100 dolares hasta 50,000 dolares por violacion, con un maximo anual de 1.5 millones de dolares por categoria de violacion.

## Cumplimiento como enfoque de gestion del riesgo

Una organizacion puede tener todas las certificaciones del mundo y seguir siendo vulnerable si trata el cumplimiento como un ejercicio burocratico. El verdadero valor de un marco de cumplimiento radica en que fuerza a la organizacion a pensar sistematicamente sobre el riesgo: que activos son criticos, que amenazas enfrentan, que controles son proporcionales y como verifican que esos controles funcionan.

El cumplimiento efectivo implica un ciclo continuo:

- **Identificacion**: que datos maneja la organizacion, que activos son criticos, que amenazas existen.
- **Evaluacion**: que tan probable es que cada amenaza se materialice y que impacto tendria.
- **Mitigacion**: que controles implementar, priorizados por riesgo residual.
- **Verificacion**: como se demuestra que los controles funcionan (auditorias, escaneos, pruebas).
- **Mejora**: que controles deben ajustarse, que procesos deben evolucionar.

Una organizacion que solo se preocupa por pasar la auditoria pierde de vista este ciclo. El resultado es un control "de papel" que existe formalmente pero no protege en la practica. Por el contrario, una organizacion que adopta el marco como herramienta de gestion del riesgo obtiene no solo el certificado sino una postura de seguridad genuinamente mas solida.

> El cumplimiento es el minimo, no el maximo. Un marco de cumplimiento establece la linea base de lo que una organizacion debe hacer. La seguridad real va mas alla, adaptandose a amenazas emergentes que los marcos quizas no cubren aun.

---

