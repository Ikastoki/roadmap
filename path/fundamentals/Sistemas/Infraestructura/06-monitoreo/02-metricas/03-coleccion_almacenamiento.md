# Coleccion y almacenamiento de metricas

Las metricas no existen hasta que alguien las recoge y las persiste. La fase de recoleccion determina que datos estan disponibles para analisis, con que precision temporal se disponen de ellos y cuantos recursos consumen en el proceso. La decision entre modelos de recoleccion (push o pull), la frecuencia de muestreo y la politica de retencion son elecciones de diseno que impactan directamente en la capacidad de detectar incidentes, en el coste de infraestructura y en la arquitectura general del sistema de monitorizacion.

## Modelo push vs pull

En el modelo push, cada aplicacion o servicio es responsable de enviar sus metricas activamente a un servidor de recogida central. La aplicacion emite una peticion HTTP, un mensaje por un bus de mensajes o una escritura directa hacia el endpoint del sistema de monitorizacion. El servidor de recogida no necesita saber nada sobre las aplicaciones: simplemente espera a que lleguen datos.

En el modelo pull, el servidor de monitorizacion es quien activamente recoge las metricas de las aplicaciones, conectandose periodicamente a un endpoint expuesto por cada servicio. La aplicacion no necesita saber que esta siendo monitorizada: solo expone un endpoint estatico (como `/metrics`) y el servidor de recogida se encarga del resto.

> El modelo pull, utilizado por Prometheus, ofrece una ventaja arquitectonica significativa: el descubrimiento automatico de targets. En entornos dinamicos donde las instancias aparecen y desaparecen constantemente (Kubernetes, autoescalado en la nube), el servidor de monitorizacion puede consultar la API del orquestador para obtener la lista actualizada de endpoints, sin que ninguna aplicacion tenga que configurar a donde enviar sus metricas.

El modelo push es preferible cuando las aplicaciones son de corta duracion (jobs batch, functions serverless) que pueden terminar antes de que el servidor de pull tenga oportunidad de recogerlas. Un job que ejecuta durante 30 segundos y luego se destruye no puede esperar a que un scraper lo visite cada 15 segundos. En estos escenarios, el job envia sus metricas durante su ejecucion o al finalizar.

Otra ventaja del push es que atraviesa firewalls de forma mas sencilla: la conexion sale del entorno de la aplicacion hacia un endpoint externo, sin necesidad de abrir puertos entrantes. El pull requiere que el servidor de monitorizacion pueda acceder a la red de las aplicaciones, lo cual puede ser problematico en entornos con segmentacion de red estricta.

Sin embargo, el push introduce un acoplamiento que puede ser indeseable. La aplicacion debe conocer la direccion del servidor de metricas, debe gestionar reintentos si el servidor no esta disponible y debe implementar logica de bufferizacion. En el pull, la aplicacion es completamente pasiva: expone datos y punto. Si el scraper no recoge las metricas durante un periodo, el problema es del sistema de monitorizacion, no de la aplicacion.

## Scraping: el ciclo de recoleccion

En el modelo pull, el proceso de scraping funciona como un ciclo continuo. Un componente scraper dentro del servidor de monitorizacion mantiene una lista de targets (endpoints) y, cada cierto intervalo, realiza una peticion HTTP GET a cada uno. La respuesta contiene las metricas en formato de texto plano o protobuf, que el servidor parsea y almacena.

El intervalo de scraping determina la resolucion temporal minima de los datos. Un intervalo de 15 segundos significa que no se podra observar un pico de latencia que dure 5 segundos y se resuelva antes de la siguiente muestra. Este es un compromiso fundamental: mayor frecuencia de scraping da mejor resolucion temporal pero genera mas carga de red y mas series temporales que almacenar.

> Un error clasico es confundir la frecuencia de scraping con la frecuencia de las alertas. Si el scraping ocurre cada 15 segundos, una metrica solo podra reflejar el estado cada 15 segundos. Una regla de alerta que evalua cada minuto esta usando cuatro muestras del scraper, no datos "en tiempo real".

El scraping tambien debe gestionar la consistencia de los datos. Si un servicio se despliega durante un scraping, el scraper puede obtener metricas parciales del antiguo proceso y parciales del nuevo. Los sistemas maduros implementan mecanismos para detectar y mitigar estas anomalias de transicion.

## Resolucion temporal (sampling frequency)

La resolucion temporal se refiere a la frecuencia con la que se registra un valor de la metrica. No es lo mismo que la frecuencia de scraping: incluso con un scraping cada 15 segundos, se puede configurar el sistema para que agrega los datos a intervalos de 1 minuto o 5 minutos antes de almacenarlos permanentemente.

La resolucion temporal afecta directamente la capacidad de detectar eventos transitorios. Un pico de latencia de 2 segundos que ocurre entre dos muestras de 30 segundos simplemente desaparece de los datos historicos. Es como intentar fotografiar un rayo con una camara que dispara una foto cada minuto: la probabilidad de capturar el evento es minima.

Sin embargo, mayor resolucion implica mas datos almacenados, mas tiempo de consulta y mayor coste. Un servicio que emite 50 metricas con un intervalo de 10 segundos genera 432.000 puntos de datos por dia. Multiplicado por 100 servicios y retenidos durante 90 dias, el volumen puede ser enorme.

La estrategia comun es mantener alta resolucion para datos recientes (ultimas horas o dias) y agregar progresivamente los datos antiguos a menor resolucion. Esto se conoce como downsampling o agregacion temporal: los datos de hace un mes pueden almacenarse a intervalos de 5 minutos en lugar de 10 segundos, perdiendo precision pero ganando eficiencia en almacenamiento y consulta.

## Retencion: cuanto guardar

La politica de retencion determina cuanto tiempo se conservan los datos de metricas antes de ser eliminados permanentemente. Esta decision depende de tres factores: capacidad de respuesta a incidentes, requisitos de analisis historico y coste de almacenamiento.

Para deteccion y respuesta a incidentes, la retencion minima suele ser de 15 a 30 dias. Los incidentes se investigan en las horas o dias posteriores a su ocurrencia, rara vez semanas despues. Para analisis de tendencias y planificacion de capacidad, se necesitan meses o anos de datos. Para cumplimiento regulatorio, algunos sectores exigen retenciones de 7 anos o mas.

> No existe una retencion unica correcta. La practica habitual es mantener datos de alta resolucion durante 7-15 dias, datos a resolucion media durante 30-90 dias y datos agregados a baja resolucion durante 1 ano o mas. Esta estrategia equilibra la capacidad de investigacion reciente con la eficiencia de almacenamiento a largo plazo.

Los sistemas como Thanos, Cortex o Mimir extienden la retencion de Prometheus indefinidamente exportando los datos a almacenamiento object (S3, GCS, MinIO) en formato comprimido. Esto permite consultar metricas de hace anos sin mantener costosos volenes de bases de datos de series temporales.

## Compromisos entre granularidad y coste

Cada decision en la cadena de recoleccion tiene implicaciones en coste. Mayor frecuencia de scraping significa mas puntos de datos, mas serie temporales activas y mayor carga en el servidor de metricas. Mayor retencion significa mas almacenamiento. Mayor cardinalidad (mas etiquetas unicas por metrica) significa multiplicar las series temporales y explotar exponencialmente el consumo de recursos.

La cardinalidad alta es particularmente peligrosa. Una metrica como `http_requests_total` con una etiqueta `user_id` que tiene 10 millones de valores unicos genera 10 millones de series temporales. El servidor de monitorizacion debe mantener cada una en memoria y en disco, lo que puede provocar un colapso del sistema. Por eso se recomienda que las etiquetas de las metricas tengan un conjunto acotado y predecible de valores posibles.

> El equilibrio optimo depende del contexto: un servicio financiero que necesita cumplir SLAs estrictos justifica mayor granularidad y retencion; un servicio interno de bajo riesgo puede operar con datos menos detallados. La clave es que la politica de retencion y resolucion sea consciente y deliberada, no accidental.

