# Respuesta a Incidentes

Un incidente de seguridad es cualquier evento que compromete la confidencialidad, integridad o disponibilidad de los sistemas y datos de una organizacion. No se trata de una posibilidad teorica sino de una certeza estadistica: a medida que la superficie de ataque crece y las techniques de los atacantes se sofistican, la pregunta no es "si" ocurrira un incidente sino "cuando" y "que tan preparada estara la organizacion para responder". La respuesta a incidentes es la disciplina que transforma el caos de un ataque en un proceso estructurado, minimizando el impacto, preservando evidencia y restaurando la operacion normal con la menor interrupcion posible. Una respuesta desorganizada no solo amplifica el dano directo del incidente sino que genera efectos secundarios devastadores: perdida de confianza de clientes, sanciones regulatorias, costes legales y dano reputacional que puede tardar anos en repararse.

La razon mas critica para tener un plan de respuesta ANTES de que ocurra un incidente es que durante un ataque las personas estan bajo presion extrema, el tiempo es limitado y las decisiones deben tomarse con informacion incompleta. Sin un plan predefinido, cada miembro del equipo improvisa segun su criterio individual, lo que lleva a acciones contradictorias, perdida de evidencia critica, comunicaciones inadecuadas y una escalada descontrolada del dano. Un plan documentado elimina la ambiguidad: cada persona sabe exactamente que hacer, con quien comunicarse y que prioridades debe respetar desde el primer minuto.

## Fases del proceso de respuesta

### Preparacion

La preparacion es la fase mas importante y la que mas tiempo demanda, precisamente porque ocurre antes de que cualquier incidente exista. Incluye la creacion del equipo de respuesta a incidentes (CSIRT/CERT), la definicion de roles y responsabilidades, la adquisicion de herramientas de analisis forense, la configuracion de canales de comunicacion seguros y la capacitacion continua del equipo mediante simulacros.

Un CSIRT funcional requiere representantes de seguridad, operaciones, legal, comunicaciones y alta direccion. Cada rol debe tener suplentes designados y los procedimientos de escalamiento deben estar documentados y accesibles offline. Las herramientas forenses deben estar disponibles en entornos aislados que no dependan de la infraestructura potencialmente comprometida. Los simulacros (tabletop exercises) deben ejecutarse al menos trimestralmente para validar que el plan funciona y que el equipo esta preparado para actuar bajo presion.

### Deteccion y analisis

La deteccion implica identificar que un incidente esta ocurriendo o ha ocurrido, lo cual proviene de multiples fuentes: sistemas de deteccion de intrusiones (IDS/IPS), SIEM, alertas de usuarios, reportes externos, inteligencia de amenazas o anomalias en el comportamiento de la red. El analisis determina la naturaleza, alcance y severidad del incidente. Aqui es donde se clasifica el incidente por nivel de criticidad, se identifican los sistemas afectados y se evalua el potencial de impacto en el negocio.

La clasificacion por severidad sigue tipicamente una escala que determina la velocidad y los recursos de respuesta:

- **Severidad baja**: incidente aislado sin afectacion a datos sensibles o servicios criticos. Resolucion por el equipo operativo estandar.
- **Severidad media**: afectacion potencial a datos o servicios, pero contenedor dentro de un perimeter controlado. Activacion parcial del CSIRT.
- **Severidad alta**: compromiso activo de sistemas criticos, datos sensibles expuestos o en riesgo, o impacto significativo en operaciones. Activacion completa del CSIRT y escalamiento a la direccion.
- **Severidad critica**: brecha masiva de datos, ransomware en produccion, compromiso de multiples sistemas, o impacto en la seguridad fisica. Activacion de protocolos de crisis con involucracion de asesoria legal externa, fuerzas del orden y comunicacion publica.

### Contencion

El objetivo de la contencion es detener la propagacion del incidente sin destruir evidencia. Esta tension entre actuar rapido y preservar evidencia es una de las decisiones mas delicadas en la respuesta a un incidente. Un borrado apresurado de logs o una reinstallacion precipitada de sistemas puede eliminar pruebas criticas para la investigacion forense, la responsabilidad legal y la prevencion futura.

La contencion tipicamente se divide en dos estrategias: contencion a corto plazo (aislamiento inmediato del sistema afectado, bloqueo de IPs maliciosas, deshabilitacion de cuentas comprometidas) y contencion a largo plazo (aplicacion de parches temporales, reconfiguracion de reglas de firewall, implementacion de compensaciones que permitan mantener la operacion con menor riesgo).

> Un error comun es confundir la contencion con la erradicacion. Contener significa limitar el dano; erradicar significa eliminar la causa. Ambas son fases distintas que pueden ocurrir en paralelo pero que no deben confundirse.

### Erradicacion

La erradicacion consiste en eliminar completamente la presencia del atacante de la infraestructura. Esto incluye la eliminacion de malware, la remocion de cuentas de backdoor, la limpieza de sistemas comprometidos y la verificacion de que no existen artefactos del atacante en otros sistemas que pudieran haber sido comprometidos durante la fase de propagacion. La erradicacion es el momento en que se aplica la leccion aprendida del incidente: los parches que no se instalaron a tiempo, las configuraciones que se toleraron como inseguras, las cuentas sin autenticacion de doble factor.

### Recuperacion

La recuperacion restaura los sistemas a su estado operativo normal, verificando que estan limpios, parchados y funcionando correctamente antes de reconnectarlos a la red de produccion. Incluye la restauracion de datos desde copias de seguridad verificadas (que no esten comprometidas), la monitorizacion intensiva de los sistemas restaurados para detectar cualquier actividad sospechosa residual y la validacion de que los controles corregidos estan efectivamente funcionando. La velocidad de recuperacion debe equilibrarse con la necesidad de certeza: restaurar un sistema comprometido sin verificar su integridad puede permitir al atacante reinsertarse.

## Comunicacion durante el incidente

La comunicacion es un componente frecuentemente subestimado en la respuesta a incidentes. Existen multiple audiencias que requieren informacion oportuna y adecuada: el equipo tecnico involucrado en la respuesta, la alta direccion que debe tomar decisiones de negocio, los equipos legales que evaluan responsabilidades, las autoridades regulatorias que deben ser notificadas segun plazos establecidos, y los clientes o usuarios cuyos datos pudieran haberse visto afectados.

Cada audiencia necesita un tipo de comunicacion distinto. El equipo tecnico necesita detalles tecnicos especificos (IPs, hashes, IOC). La direccion necesita una vision de impacto en el negocio (que servicios estan afectados, cual es el timeline estimado de recuperacion, que riesgos financieros existen). Legal necesita una evaluacion de obligaciones de notificacion y responsabilidad. Los clientes necesitan una comunicacion clara sobre que les afecta, que esta haciendo la organizacion y que medidas deben tomar ellos.

> La transparencia durante un incidente es un activo reputacional. Las organizaciones que comunican abiertamente sobre un incidente (dentro de lo que la situacion permite) recuperan la confianza mucho mas rapido que las que intentan minimizar o ocultar el dano.

## Lecciones aprendidas

Despues de cada incidente, el equipo de respuesta debe realizar un post-mortem o revision lecciones aprendidas. Este proceso examina que funciono, que fallo, que se hizo bien y que debe mejorar. No es un ejercicio de culpabilizacion sino de mejora continua. Los hallazgos se documentan formalmente y se traducen en acciones concretas: actualizaciones al plan de respuesta, nuevas herramientas, cambios en la arquitectura, mejoras en la monitorizacion o ajustes en la formacion del equipo.

El post-mortem debe ser exhaustivo y honesto. Si un contenedor fallo porque un sistema no tenia las copias de seguridad adecuadas, la causa raiz no es "el atacante era muy sofisticado" sino "no se verificaron las copias de seguridad regularmente". Si la deteccion tardo 72 horas, la pregunta no es "por que no nos dimos cuenta" sino "que monitoreo falta para reducir el tiempo de deteccion a horas en lugar de dias".

Cada incidente es una oportunidad para fortalecer la postura de seguridad de la organizacion. Las organizaciones que aprenden de sus incidentes se vuelven progresivamente mas resistentes; las que los olvidan o los minimizan estan condenadas a repetir los mismos errores con consecuencias cada vez mayores.

---

