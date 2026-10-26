# Herramientas de Tracing

El tracing distribuido resuelve uno de los problemas mas complejos de la observabilidad en arquitecturas de microservicios: entender que le ocurre a una unica peticion cuando atraviesa multiples servicios, colas de mensajes y bases de datos. Sin tracing, un equipo de operaciones solo puede ver metricas agregadas por servicio, lo que dificulta enormemente identificar que componente concreto introduce latencia o errores en un flujo end-to-end. Las herramientas de tracing distribuido capturan la jerarquia de spans (unidades de trabajo) que componen una peticion, permitiendo reconstruir el camino completo de la solicitud y localizar el punto exacto de fallo o cuello de botella.

## Jaeger: tracing de extremo a extremo

Jaeger, desarrollado originalmente por Uber y posteriormente donado a la CNCF como proyecto graduated, implementa la especificacion OpenTracing para tracing distribuido. Su arquitectura se compone de un agente que recibe spans desde las aplicaciones, un collector que procesa y valida los datos, y un backend que almacena las trazas en Elasticsearch, Cassandra o Kafka para durabilidad y escalabilidad.

El concepto fundamental de Jaeger es el span: cada operacion dentro de un servicio (una llamada HTTP, una consulta a base de datos, una operacion de negocio) se representa como un span con un tiempo de inicio, un tiempo de fin, un identificador unico y una referencia al span padre que lo originó. Cuando una peticion viaja de servicio A a servicio B, el span creado en A se convierte en el padre del span creado en B, estableciendo una jerarquia que refleja exactamente la topologia de la llamada.

Jaeger ofrece dos modos de instrumentacion: manual, donde el desarrollador instrumenta explicitamente el codigo con llamadas a la API de tracing, y automatica, mediante librerias de instrumentacion que interceptan automaticamente las llamadas HTTP, gRPC y operaciones de bases de datos. El modo automatico es fundamental en microservicios donde instrumentar manualmente cada servicio es inviable. Las librerias de instrumentacion automatica propagan el contexto de tracing a traves de los headers HTTP (tipicamente `uber-trace-id` o el estandar `traceparent` de W3C), permitiendo que la traza se mantenga intacta incluso cuando la peticion atraviesa colas de mensajes asincronos.

> Jaeger Query proporciona una interfaz grafica donde se pueden buscar trazas por servicio, duracion, operation name y tags, visualizando el diagrama de spans como un arbol jerarquico con informacion de timing en cada nodo.

La componente de sampling es critica en Jaeger porque capturar el 100% de las peticiones en un sistema con alto throughput generaria un volumen de datos ingestion que saturaria el almacenamiento y la red. Jaeger implementa estrategias de sampling configurables por servicio: sampling constante (capturar 1 de cada N peticiones), probabilistico (capturar cada peticion con una probabilidad P) y rate limiting (capturar un maximo de N trazas por segundo). El sampling adaptativo, disponible en configuraciones avanzadas, ajusta automaticamente la tasa de muestreo basandose en el comportamiento del trafico, capturando mas trazas durante picos inusuales.

## Zipkin: el pionero del tracing distribuido

Zipkin, originado en Twitter como implementacion del paper de Google "Dapper", es una de las herramientas de tracing distribuido mas longevas del ecosistema. Su arquitectura es conceptualmente similar a Jaeger: colectores que reciben spans, un almacenamiento backend (Cassandra, Elasticsearch o MySQL) y un servidor web que proporciona la interfaz de busqueda y visualizacion.

La principal diferencia operativa entre Zipkin y Jaeger radica en la comunidad y la adopcion. Zipkin tiene una base de usuarios mas amplia en ecosistemas JVM gracias a su integracion nativa con Spring Cloud, mientras que Jaeger tiene mayor traccion en entornos Kubernetes y Cloud Native. Zipkin tambien implementa su propio protocolo de propagacion de contexto, aunque ambos soportan el header estandar `B3` de propagacion.

Zipkin ofrece instrumentacion automatica mediante Brave, su libreria de instrumentacion para Java, y Zipkin Reporter, que envia los spans al collector. Para lenguajes no-JVM, el ecosistema de contribuciones comunitarias proporciona clientes en Python, Go, Node.js y Rust, aunque con menor madurez que el ecosistema Java.

En la practica, la eleccion entre Jaeger y Zipkin rara vez depende de diferencias tecnicas significativas, ya que ambos implementan funcionalidades comparables. La decision se inclina tipicamente por Jaeger en entornos nuevos que buscan alinearse con el ecosistema CNCF, y por Zipkin en organizaciones con inversion existente en Spring Cloud y ecosistema JVM.

## OpenTelemetry: la convergencia de las tres pilas

OpenTelemetry (OTel) no es una herramienta de tracing, sino un framework de instrumentacion y coleccion que unifica metricas, logs y trazas bajo una unica especificacion y API estandar. Nacido de la fusion de OpenTracing (estandar de tracing) y OpenCensus (estandar de metricas y tracing) en 2019, OpenTelemetry se ha convertido en el segundo proyecto con mas contribuidores de la CNCF despues de Kubernetes, lo que refleja su importancia estrategica para la industria.

El proposito fundamental de OpenTelemetry es eliminar el acoplamiento entre la instrumentacion de la aplicacion y el backend de observabilidad. Sin OTel, una aplicacion que usa el SDK de Jaeger queda vinculada al protocolo de Jaeger; si la organizacion decide migrar a Zipkin o a Datadog, debe re-instrumentar la aplicacion. Con OpenTelemetry, la aplicacion instrumenta usando la API estandar de OTel y exporta los datos a traves de exporters que pueden cambiarse sin modificar una sola linea de codigo de la aplicacion.

OpenTelemetry define un SDK por lenguaje que proporciona APIs para crear spans, metricas y eventos de log, asi como un Collector que actua como proxy configurable que recibe datos en el formato OTel y los enruta a multiples backends simultaneamente. Este patron de "instrumentar una vez, exportar a cualquier lugar" es el valor propuesto central de OTel.

```go
tracer := otel.Tracer("mi-servicio")
ctx, span := tracer.Start(ctx, "procesar-pedido")
defer span.End()
```

> El ejemplo anterior crea un tracer y un span usando la API estandar de OpenTelemetry. El mismo codigo funciona sin modificaciones si se cambia el backend de Jaeger a Datadog, solo cambiando el exporter en la configuracion del colector.

El OpenTelemetry Collector soporta receivers (reciben datos en multiples formatos), processors (transforman, muestrean y enriquecen los datos) y exporters (envian datos a multiples destinos). Esta arquitectura de pipeline modular permite configurar un unico agente de coleccion que alimente Prometheus para metricas, Loki para logs, Jaeger para trazas y simultaneamente un backend comercial como Datadog, todo sin duplicar la instrumentacion.

La razon por la que OpenTelemetry se esta consolidando como estandar es triple. Primero, es vendor-neutral: no esta controlado por ningun proveedor comercial, lo que elimina el riesgo de lock-in. Segundo, es un proyecto CNCF con el respaldo de la comunidad cloud native, lo que garantiza soporte a largo plazo y adopcion masiva. Tercero, el modelo de "instrumentar una vez" resuelve el problema mas doloroso de la observabilidad: el coste de re-instrumentar aplicaciones cuando cambia la herramienta de backend. Las organizaciones que adoptan OpenTelemetry hoy estan protegidas contra cambios futuros en su stack de observabilidad.

> Para metricas, OpenTelemetry expone datos en el formato OTLP que puede ser ingerido directamente por Prometheus (desde la version 2.47) o por backends como Grafana Mimir y Cortex, eliminando la necesidad de usar el Prometheus client library nativo en cada lenguaje.

