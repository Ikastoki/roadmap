# SLI, SLO y SLA

La fiabilidad de un servicio no se puede gestionar con intuiciones ni promesas vagas. Las organizaciones necesitan un lenguaje comun y preciso para medir, objetivar y comprometer el nivel de rendimiento que entregan. SLI, SLO y SLA forman un marco conceptual escalonado donde cada nivel construye sobre el anterior: el SLI mide lo que realmente ocurre, el SLO establece lo que se aspira a conseguir, y el SLA formaliza lo que se promete contractualmente. Sin este marco, las decisiones sobre fiabilidad se toman de forma reactiva, bajo presion, y con consecuencias economicas impredecibles.

## SLI: Service Level Indicator (Indicador de Nivel de Servicio)

Un SLI es una metrica concreta y cuantificable que refleja el comportamiento real del servicio desde la perspectiva del usuario. No es cualquier metrica del sistema; es una seleccion deliberada de aquella que captura la experiencia directa de quien consume el servicio. Los SLIs mas comunes son: la fraccion de peticiones exitosas (*availability*), la latencia de respuesta (*performance*), la integridad de los datos (*correctness*), y la duracion de las interrupciones (*durability*).

La clave de un buen SLI es que sea observable y medible de forma continua. "El servicio esta rapido" no es un SLI; "el percentil 95 de latencia de la API de usuarios es de 380ms" si lo es. Los SLIs se derivan de la telemetria que ya se recolecta (metricas, traces, logs), pero no toda telemetria califica como SLI. Un operador podria monitorizar la temperatura del CPU o la tasa de garbage collection, pero estas son metricas de infraestructura que no capturan directamente la experiencia del usuario. El SLI debe responder a la pregunta: "si este numero esta mal, al usuario le importa".

La seleccion de SLIs varia segun el tipo de servicio. Un servicio de procesamiento de pagas podria utilizar como SLI la fraccion de pagas procesadas correctamente antes de la fecha de vencimiento. Un servicio de streaming video utilizaria la fraccion de sesiones donde la reproduccion se mantuvo sin interrupciones. Lo que todos tienen en comun es que son metricas que, si empeoran, justifican una inversion en fiabilidad.

## SLO: Service Level Objective (Objetivo de Nivel de Servicio)

Un SLO es el valor objetivo interno que una equipo se compromete a mantener para un SLI dado. Si el SLI es "que esta pasando", el SLO es "que queremos que pase". Por ejemplo, si el SLI mide la disponibilidad efectiva del servicio como el 99.95% de las peticiones exitosas en un periodo de 30 dias, el SLO podria ser mantener esa disponibilidad por encima del 99.9%.

Los SLOs son decisiones de ingenieria, no compromisos comerciales. Definir un SLO implica un balance consciente entre fiabilidad y velocidad de desarrollo. Un SLO de 99.99% (cuatro nueves) requiere una inversion significativamente mayor en infraestructura redundante, testing automatizado y procesos de despliegue que un SLO de 99.9% (tres nueves). Cada nueve adicional reduce exponencialmente el margen de error disponible, lo que se traduce en mayor complejidad tecnica y coste operativo.

> Un SLO no es un numero magico que se elige por que suena bien. Es el resultado de una negociacion interna entre el equipo de producto, ingenieria y operaciones, basada en las consecuencias reales de no cumplirlo.

El concepto de *error budget* (presupuesto de errores) se deriva directamente del SLO. Si el SLO es 99.9% de disponibilidad, el error budget es el 0.1% de tiempo o peticiones que el equipo puede "gastar" sin consecuencias. Este presupuesto se consume con cada incidente, cada despliegue con downtime y cada degradacion de rendimiento. Cuando el error budget se agota, el equipo debe priorizar fiabilidad sobre nuevas funcionalidades. Cuando el error budget esta saludable, el equipo tiene libertad para experimentar y desplegar con mas frecuencia. Esta mecanica crea un equilibrio natural entre velocidad y estabilidad.

## SLA: Service Level Agreement (Acuerdo de Nivel de Servicio)

Un SLA es un compromiso contractual formal entre el proveedor del servicio y el cliente. A diferencia del SLO, que es interno, el SLA tiene consecuencias legales y economicas. Si el SLA establece una disponibilidad del 99.9% y el servicio no lo cumple, el proveedor debe compensar al cliente, generalmente con credito de servicio o descuentos futuros.

El SLA no debe confundirse con el SLO. Un SLO puede ser mas estricto que el SLA, porque un equipo inteligente no quiere operar justo en el limite de su compromiso contractual. La diferencia entre ambos es el margen de seguridad: si el SLA promete 99.9%, el SLO interno podria ser 99.95%, dando al equipo un colchon de 0.05% antes de incurrir en penalizaciones.

Un SLA bien disenado define no solo la metrica y el objetivo, sino tambien las condiciones de medicion (que se excluye: mantenimientos programados, fuerza mayor), el metodo de verificacion (que herramientas usan ambas partes), y las consecuencias del incumplimiento (que penalizaciones aplican y como se calculan). La ambiguedad en estos terminos es la fuente principal de disputas contractuales en servicios de tecnologia.

```
# Jerarquia de fiabilidad
SLI  --> Que medimos (latencia, disponibilidad, errores)
SLO  --> Que aspiramos (99.9% de disponibilidad interna)
SLA  --> Que prometemos (99.9% contractual con penalizaciones)
```

## La relacion completa: del indicador al compromiso

El flujo completo funciona como una cadena de traduccion. Primero, se identifica que le importa al usuario (disponibilidad, rapidez, correccion) y se codifica en un SLI medible. Luego, se establece un SLO que define el nivel de servicio aceptable basado en la tolerancia al riesgo y los recursos disponibles. Finalmente, si existe un compromiso externo, el SLA traduce ese SLO a terminos legales, usualmente con un margen conservador.

El error budget actua como el regulador de este sistema. Cuando el equipo consume su error budget rapidamente (muchos incidentes, despliegues problemáticos), recibe una senal clara de que debe invertir en fiabilidad antes de agregar nuevas funcionalidades. Cuando el error budget esta intacto, tiene licencia para innovar con mas agresividad. Este ciclo crea un proceso de mejora continua donde la fiabilidad no es una preocupacion estatica, sino un recurso gestionado activamente.

La trampa mas comun es ignorar el error budget cuando esta lleno, tratandolo como un indicador irrelevante. Los equipos que despliegan funcionalidades sin importar la fiabilidad eventualmente agotan su presupuesto de errores, y entonces se ven obligados a detener todo el desarrollo para estabilizar el sistema. Este patron de *boom and bust* (auge y colapso) es la señal de que el proceso de gestion de fiabilidad no se esta siguiendo con disciplina.

