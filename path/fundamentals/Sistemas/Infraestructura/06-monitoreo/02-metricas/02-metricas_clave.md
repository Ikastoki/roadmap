# Metricas clave

Los sistemas distribuidos modernos generan miles de series temporales potenciales: latencia de cada endpoint, uso de memoria por contenedor, tasa de errores por tipo de peticion, conexiones TCP activas, tamano de colas. Sin un marco de referencia estructurado, un ingeniero puede pasar horas observando graficos sin llegar a conclusiones accionables. Las Cuatro Senales Doradas (Four Golden Signals), definidas por el libro de Google sobre ingenieria de fiabilidad del site (SRE), ofrecen un modelo minimalista pero extraordinariamente efectivo que cubre la vasta mayoria de problemas de salud de un servicio. La premisa es que si monitorizas solo cuatro dimensiones, detectaras la inmensa mayoria de degradaciones antes de que afecten a los usuarios de forma significativa.

## Latencia

La latencia es el tiempo que transcurre desde que un sistema recibe una peticion hasta que devuelve una respuesta. No es un numero unico sino una distribucion: cada peticion tiene su propia duracion, y lo que importa es entender como se distribuyen esos tiempos a traves de toda la poblacion de peticiones.

Medir solo la latencia promedio es una practica peligrosa. Un servicio que atiende 1.000 peticiones por segundo con una latencia promedio de 50ms puede tener 10 peticiones que tardan 30 segundos cada una, afectando a usuarios reales mientras el promedio sigue luciendo saludable. Por eso, la latencia siempre debe monitorizarse con percentiles: P50 (mediana), P95, P99 y P99.9. El percentil 99 representa el tiempo de respuesta del 1% de las peticiones mas lentas, que en servicios con alto volumen equivale a miles de usuarios.

La latencia debe separarse entre peticiones exitosas y fallidas. Una peticion que falla con un error 500 y responde en 1ms no es "rapida": simplemente aborto prematuramente. Las metricas de latencia solo son representativas sobre el conjunto de respuestas con codigo 2xx o 3xx.

> Una analogia util: si un semaforo tarda 2 segundos en cambiar de verde a rojo en promedio pero cada 100 cambios tarda 30 segundos, los conductores lo notaran inmediatamente aunque el promedio sea "bueno". Los percentiles altos capturan estos picos que el promedio oculta.

## Trafico (Traffic)

El trafico representa el volumen de trabajo que el sistema esta procesando. La metrica mas comun es el numero de peticiones por segundo (requests per second o RPS), pero el trafico puede medirse en bytes transferidos, operaciones de base de datos ejecutadas, jobs procesados de una cola o cualquier otra unidad de trabajo relevante para el servicio.

El trafico es el indicador de demanda del sistema. Comparar el trafico actual con el historico permite distinguir entre una caida de rendimiento (el sistema se ha degradado con la misma carga) y un sobrecargo (el sistema funciona correctamente pero recibe mas trafico del que puede manejar). Esta distincion es critica para la accion correctiva: en el primer caso se busca un bug o cuello de botella; en el segundo, se escalan recursos.

Monitorizar el trafico tambien permite detectar anomalias de seguridad. Un pico repentino de trafico en un endpoint que normalmente recibe poca actividad puede indicar un ataque DDoS, un escaneo de vulnerabilidades o un cliente mal configurado que envia peticiones en un bucle infinito.

> En arquitecturas de microservicios, el trafico de cada servicio debe medirse de forma independiente. Un gateway puede recibir 10.000 RPS pero el servicio de autenticacion que invoca solo recibe 3.000. El trafico del gateway no refleja la carga del servicio interno.

## Errores (Errors)

La tasa de errores es la proporcion de peticiones que terminan con un resultado incorrecto respecto al total de peticiones procesadas. Definir que constituye un "error" es el paso mas critico y frecuentemente mal ejecutado. Un error 500 es claramente un fallo, pero un timeout que devuelve 200 con un body vacio tambien lo es. Una respuesta 200 que contiene datos corruptos o incompletos es un error silencioso, el tipo mas peligroso.

La tasa de errores se calcula como el numero de peticiones con resultado erroneo dividido por el total de peticiones en un intervalo de tiempo. Esta tasa es mas informativa que el conteo absoluto de errores porque normaliza frente al volumen de trafico. 100 errores en un segundo con 10.000 peticiones (1%) es muy diferente a 100 errores en un segundo con 150 peticiones (66%).

Los errores deben clasificarse por tipo, endpoint y severidad. No todos los errores son iguales: un 404 en un recurso inexistente puede ser trafico legitimo con una URL malformada, mientras que un 503 en el servicio de pagos es una emergencia. La taxonomia de errores permite crear alertas diferenciadas y priorizar la resolucion.

> Un patron comun de fallo silencioso es el "error envelope": un servicio empieza a devolver respuestas parcialmente correctas. Los logs muestran pocos errores, los dashboards de tasa de error parecen normales, pero los usuarios reportan datos inconsistentes. Esto se detecta monitorizando la integridad de las respuestas, no solo sus codigos de estado.

## Saturacion (Saturation)

La saturacion mide cuanto de los recursos disponibles del sistema estan siendo utilizados. Es el indicador predictivo por excelencia: cuando un recurso esta al 90% de utilizacion, el servicio puede seguir funcionando correctamente, pero esta al borde de una degradacion. La saturacion permite anticipar problemas antes de que se materialicen en latencia elevada o errores.

Los recursos habituales a monitorizar incluyen CPU, memoria, disco, red, numero de conexiones a base de datos y profundidad de colas de mensajes. Cada servicio tiene sus propios recursos criticos: un proxy inverso se satura en conexiones concurrentes, un servicio de procesamiento de imagenes se satura en CPU y memoria, un servicio de cache se satura en RAM.

> La relacion entre saturacion y latencia no es lineal. Un sistema puede operar establemente al 70% de CPU y de repente experimentar un aumento exponencial de latencia al 85% debido a la contention por recursos, la activacion de mecanismos de swap o la llegada de peticiones de latencia alta que desplazan a las rapidas de las colas de procesamiento. Este comportamiento se conoce como "cliff effect" y es la razon por la que las alertas de saturacion deben activarse con margen.

## Por que solo cuatro senales

Las cuatro senales forman un modelo causal: el trafico genera demanda, la demanda consume recursos (saturacion), la saturacion degrada la latencia, y la degradacion de la latencia puede provocar errores (por timeouts, por ejemplo). Monitorizar las cuatro permite trazar el problema desde su causa raiz hasta su efecto en el usuario.

Un sistema que monitoriza solo la latencia sabra que algo va mal pero no sabra si la causa es un pico de trafico, un recurso saturado o un bug que genera errores. Un sistema que monitoriza solo los errores detectara fallos pero no sabra si son ocasionales (y por tanto probablemente debidos a un bug puntual) o sistematicos (y por tanto debidos a saturacion o degradacion de dependencias).

Las Cuatro Senales no cubren todos los escenarios: no miden la consistencia de datos, la seguridad del sistema ni la experiencia subjetiva del usuario. Pero para el 80% de los problemas de operaciones, estas cuatro dimensiones proporcionan suficiente informacion para diagnosticar y resolver el incidente de forma efectiva.

