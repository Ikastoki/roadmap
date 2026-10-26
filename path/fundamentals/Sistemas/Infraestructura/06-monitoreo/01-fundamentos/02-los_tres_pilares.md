# Los Tres Pilares de la Observabilidad

Un sistema observable no depende de una sola fuente de datos para entender su comportamiento. Si solo se recogen metricas, se pierde contexto sobre que eventosspecificos causaron una anomalia. Si solo se registran logs, no se tiene una vision agregada de tendencias. Si solo se rastrean requests, se carece de la perspectiva de salud general del sistema. La observabilidad efectiva se construye sobre tres pilares complementarios que, juntos, permiten reconstruir el estado interno del sistema desde diferentes angulos: metricas, logs y traces. Cada uno responde a una pregunta distinta, y su poder real emerge cuando se correlacionan entre si.

## Metricas: el "que" esta pasando

Las metricas son valores numericos medidos en el tiempo que representan el estado cuantificable de un sistema. Son la forma mas eficiente de almacenar y consultar tendencias:CPU en uso, numero de requests por segundo, latencia de respuesta, tasa de errores, consumo de memoria. Su naturaleza compacta y estructurada las hace ideales para dashboards, alertas y analisis de tendencias a largo plazo.

La fortaleza de las metricas esta en su capacidad de sintesis. Con una sola grafica de latencia P99 se puede entender si un servicio esta degradando su rendimiento. Con una tasa de errores en increasing se puede detectar un despliegue defectuoso. Las metricas responden a la pregunta fundamental: **que** esta ocurriendo en el sistema en terminos agregados.

Sin embargo, las metricas por si solas son ciegas a la causa. Si la latencia sube, la metrica lo indica, pero no explica si es por una consulta a base de datos lenta, un bloqueo de concurrencia o un problema de red. Las metricas son el sintoma; no son el diagnostico.

> Las metricas son como las senales vitales de un paciente: temperatura, presion arterial, frecuencia cardiaca. Indican que algo va mal pero no dicen que enfermedad tiene el paciente.

## Logs: el "cuando" y "que paso"

Los logs son registros discretos de eventos que ocurren en un momento especifico dentro del sistema. Cada linea de log representa un hecho: una peticion HTTP recibida, una transaccion completada, un error capturado, una decision tomada por el algoritmo de balanceo de carga. A diferencia de las metricas, los logs preservan el contexto detallado de cada evento individual.

Los logs son la memoria narrativa del sistema. Mientras las metricas dicen "hubo un aumento de errores", los logs dicen "a las 14:32:07, el servicio de pagos recibio una peticion con un token de autenticacion expirado, reintentó con el secundario, y este tambien rechazo la peticion porque la base de datos de sesiones estaba temporalmente no disponible". Esta riqueza contextual es insustituible para el diagnostico profundo.

El desafio de los logs radica en su volumen y en la dificultad de correlacionarlos. En un sistema distribuido con docenas de servicios, un solo request puede generar cientos de lineas de log repartidas en multiples maquinas. Buscar la causa de un fallo implica navegar millones de lineas de texto, lo cual es inviable sin herramientas de agregacion y busqueda.

> Los logs son como el historial clinico completo de un paciente: cada analisis, cada consulta, cada sintoma reportado. Son exhaustivos pero requieren un medico (o un motor de busqueda) que sepa que buscar y como interpretarlos en conjunto.

## Traces: el "como fluye" la informacion

Los traces (rastros o trazas) son el registro del recorrido completo de una peticion a traves de multiples servicios y componentes del sistema. Un trace comienza cuando un usuario realiza una accion (por ejemplo, comprar un producto) y sigue cada paso: la carga del balanceador, la autenticacion en el gateway, la consulta al servicio de inventario, la escritura en la base de datos de pedidos, la llamada al servicio de notificaciones. Cada paso se registra como un span, y todos los spans se agrupan bajo un trace ID unico.

Los traces responden a la pregunta: **como** fluye una peticion a traves del sistema y en que punto exacto se produce la latencia o el error? Un trace bien instrumentado permite ver, por ejemplo, que un request que toma 3 segundos pasa 2.8 segundos esperando una respuesta de una base de datos, mientras que el resto de la cadena se ejecuta en milisegundos. Esta informacion es inmediatamente accionable: el problema no es el servicio en si, sino su interaccion con la base de datos.

En arquitecturas de microservicios, los traces son indispensables. Sin ellos, entender como interactuan decenas de servicios en una sola operacion de usuario es practicamente imposible. Los traces convierten la complejidad de una arquitectura distribuida en un diagrama secuencial legible que muestra exactamente donde se invierte el tiempo.

> Los traces son como el flujo de sangre a traves del cuerpo: permiten seguir el camino desde el corazon hasta cada organo, identificando exactamente donde hay una obstruccion o una fuga que no se detectaria solo mirando la presion arterial (metricas) o el historial clinico (logs).

## Como se complementan

El verdadero poder de la observabilidad emerge cuando los tres pilares se correlacionan. El flujo tipico de diagnostico funciona asi:

1. **Metricas** detectan la anomalia: la tasa de errores HTTP 5xx ha subido del 0.1% al 5% en los ultimos 15 minutos.
2. **Logs** revelan el contexto: los errores son todos "Connection refused" hacia el servicio de recomendaciones.
3. **Traces** localizan la causa: el trace muestra que el servicio de recomendaciones tarda 10 segundos en responder cuando la carga supera las 1000 requests concurrentes.

Sin las metricas, el problema podria pasar desapercibido durante horas. Sin los logs, no se sabria a que servicio afecta. Sin los traces, no se identificaria que el cuello de botella esta en un componente especifico y no en la cadena completa.

Cada pilar tiene fortalezas y debilidades que compensan las de los otros. Las metricas son eficientes en almacenamiento pero carecen de contexto. Los logs son ricos en contexto pero costosos en volumen. Los traces muestran la relaciones entre componentes pero requieren instrumentacion en cada servicio. La combinacion de los tres crea una imagen completa que ninguno podria proporcionar por separado.

> La analogia medica es precisa: un medico que solo tiene las senales vitales (metricas) puede detectar una emergencia pero no saber la causa. Un medico que solo tiene el historial clinico (logs) puede ver patrones pasados pero no la situacion actual en tiempo real. Un medico que solo tiene estudios de flujo sanguineo (traces) puede ver la circulacion pero no el contexto clinico. Los tres juntos permiten un diagnostico preciso y accionable.

