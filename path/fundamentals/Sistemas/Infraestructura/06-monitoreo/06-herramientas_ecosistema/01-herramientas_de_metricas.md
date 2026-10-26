# Herramientas de Metricas

Las metricas constituyen la columna vertebral de cualquier sistema de observabilidad, pero su valor real solo se materializa cuando existen herramientas capaces de recopilar, almacenar y consultar eficientemente grandes volumenes de datos temporales. Un sistema de metricas no es simplemente un repositorio de numeros: es una maquina de indexing que permite responder preguntas como "cual fue la tasa de errores en la ultima hora por servicio?" en milisegundos. La eleccion de la herramienta de metricas determina no solo el rendimiento de las consultas, sino tambien el coste operativo del sistema de monitoreo a largo plazo, ya que las series temporales crecen de forma exponencial con la cardinalidad.

## Prometheus: el modelo pull como paradigma

Prometheus, creado por SoundCloud y ahora proyecto graduated de la CNCF, adopto una arquitectura radicalmente distinta a los sistemas de monitoreo tradicionales: en lugar de que las aplicaciones empujen (push) sus metricas hacia un servidor central, Prometheus extrae (pull) las metricas de los endpoints de las aplicaciones en intervalos regulares. Este modelo pull ofrece ventajas arquitectonicas significativas. En primer lugar, Prometheus es quien controla la frecuencia de muestreo, lo que elimina la posibilidad de que un cliente envie metricas a un ritmo que sature el servidor. En segundo lugar, si una instancia esta caida, Prometheus lo detecta automaticamente porque el endpoint no responde, convirtiendo la ausencia de datos en senal util. En tercer lugar, el pull model simplifica la topologia de red: las aplicaciones no necesitan conocer la direccion del servidor de monitoreo, solo exponer un endpoint HTTP estandar.

El motor de consultas de Prometheus es PromQL, un lenguaje declarativo especialmente disenado para series temporales. PromQL permite seleccionar series, filtrar por etiquetas, realizar operaciones aritmeticas entre series, aplicar funciones de agregacion (rate, histogram_quantile, increase) y combinar resultados de multiples consultas. Su poder radica en que trabaja directamente con el concepto de "rango de tiempo", permitiendo calcular tasas, medias moviles y percentiles sobre ventanas temporales arbitrarias.

```
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m])
```

> La expresion anterior calcula la fraccion de respuestas 5xx sobre el total de respuestas en una ventana de 5 minutos, un patron clasico de error rate en SRE.

Prometheus utiliza un almacenamiento local basado en un time-series database optimizado para escritura_APPEND y lectura por rango. Los datos se organizan en bloques en disco, con un WAL (Write-Ahead Log) que garantiza durabilidad ante caidas inesperadas. Sin embargo, el almacenamiento local tiene limitaciones claras: no escala horizontalmente y la retencion es limitada. Para resolverlo, el ecosistema Prometheus ofrece Thanos y Cortex (ahora Mimir), que anaden replicacion, retencion a largo plazo y consulta federada sobre multiples instancias.

La arquitectura de Prometheus incluye un componente de descubrimiento de targets que se integra nativamente con Kubernetes, Consul, EC2 y otros orquestadores. Este descubrimiento dinamico es fundamental en entornos modernos donde las instancias aparecen y desaparecen continuamente. Prometheus detecta automaticamente nuevos pods en Kubernetes sin necesidad de configuracion manual, asignando metricas a los labels correctos de cada instancia.

## Grafana: la lente, no el almacenamiento

Grafana es frecuentemente confundido con una plataforma de metricas, pero en realidad es una herramienta de visualizacion y correlacion que se conecta a multiples fuentes de datos. Su valor no reside en almacenar nada, sino en proporcionar una interfaz unificada para construir dashboards, establecer alertas visuales y correlacionar informacion proveniente de Prometheus, Elasticsearch, Loki, InfluxDB, bases de datos SQL y practicamente cualquier fuente con un plugin disponible.

La razon por la que Grafana se convirtio en el estandar de facto de la visualizacion es su modelo de plugins abierto y su capacidad de orquestar multiples fuentes de datos en un mismo dashboard. Un equipo puede visualizar metricas de Prometheus, logs de Loki y trazas de Jaeger en un unico panel, creando una experiencia de observabilidad unificada sin necesidad de centralizar los datos. Grafana permite definir dashboards como JSON, lo que facilita el versionado, la replicacion entre entornos y el patron "dashboard as code".

Grafana introduce el concepto de "variables" en los dashboards, que permiten crear paneles dinamicos cuyo contenido cambia segun el selector que el usuario elige. Un dashboard de Kubernetes puede tener una variable para seleccionar namespace, otra para seleccionar pod y otra para seleccionar metrica, todo interactivo y en tiempo real. Esta capacidad de drill-down es una de las razones principales de su adopcion masiva.

> Grafana Alerting (anteriormente Grafana Unified Alerting) permite definir reglas de alerta que se evaluan contra multiples fuentes de datos, integrando notificaciones con Canales de Alertmanager, Slack, PagerDuty, email y otros sistemas.

## InfluxDB y TimescaleDB: bases de datos temporales

InfluxDB es una base de datos de series temporales disenaada especificamente para metricas de alta cardinalidad y alta frecuencia de escritura. A diferencia de Prometheus, InfluxDB utiliza un modelo push: las aplicaciones envian metricas directamente a la base de datos. Esto lo hace mas adecuado para escenarios donde el emisor tiene control sobre la configuracion de envio y necesita retencion a largo plazo sin depender de componentes externos como Thanos. InfluxDB ofrece su propio lenguaje de consulta, InfluxQL, y mas recientemente Flux, un lenguaje funcional mas expresivo pero tambien mas complejo.

TimescaleDB adopta un enfoque distinto: extiende PostgreSQL con funcionalidades especificas para series temporales mediante hypertables y chunking automatico. La ventaja de este enfoque es que aprovecha todo el ecosistema de herramientas SQL, indices B-tree, extensiones de PostgreSQL y herramientas de administracion ya existentes. Para organizaciones que ya tienen infraestructura PostgreSQL y conocimiento del ecosistema, TimescaleDB elimina la necesidad de introducir un motor de almacenamiento nuevo.

La decision entre InfluxDB y TimescaleDB depende del contexto operativo. InfluxDB es mas rapido para escrituras puras de metricas y tiene un modelo de datos nativo de series temporales. TimescaleDB es mas flexible para consultas analiticas complejas y se integra mejor en entornos donde SQL es el estandar de acceso a datos.

