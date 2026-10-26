# Instrumentacion

拥有一套完善的 sistema de tracing es inutil si los servicios no generan los spans necesarios para reconstruir el viaje de las peticiones. La instrumentacion es el proceso de dotar al codigo de la capacidad de emitir telemetria de trazas de forma automatica o manual. La calidad de la instrumentacion determina directamente la utilidad del tracing: una instrumentacion incompleta genera trazas con brechas que ocultan la informacion critica, mientras una instrumentacion excesiva puede degradar el rendimiento y crear ruido que dificulta el analisis. Elegir el enfoque correcto de instrumentacion requiere equilibrar granularidad, mantenimiento y impacto en rendimiento.

## Instrumentacion manual

La instrumentacion manual implica que cada desarrollador crea explicitamente los spans en el codigo de la aplicacion, definiendo donde comienza y termina cada operacion de interes. El programador decide que partes del codigo son relevantes para el tracing, que metadatos incluir en cada span y como estructurar la jerarquia de spans.

La principal ventaja de la instrumentacion manual es el control total. Puedes crear spans que representen operaciones de negocio concretas como "procesar pedido", "validar stock" o "calcular impuestos", no solo las operaciones tecnicas como llamadas HTTP o queries SQL. Puedes anadir atributos semanticamente ricos que faciliten el filtrado y analisis posterior. Si tu sistema tiene una operacion critica que no corresponde a ninguna llamada de red, solo la instrumentacion manual puede capturarla.

```python
with tracer.start_as_current_span("procesar_pedido") as span:
    span.set_attribute("pedido.id", pedido.id)
    span.set_attribute("pedido.total", pedido.total)
    validar_stock(pedido)
    calcular_impuestos(pedido)
```

El inconveniente es claro: requiere disciplina del desarrollador, es propensa a errores humanos y añade carga de mantenimiento. Cada servicio nuevo o modificacion requiere que alguien actualice la instrumentacion. En organizaciones grandes con cientos de servicios, mantener una instrumentacion manual consistente es un desafio significativo.

## Instrumentacion automatica

La instrumentacion automatica utiliza bibliotecas o agentes que interceptan las llamadas a frameworks y librerias conocidas para generar spans de forma transparente al desarrollador. Sin modificar una sola linea de codigo de la aplicacion, estas herramientas instrumentan automaticamente las llamadas HTTP salientes, las conexiones a bases de datos, las operaciones sobre colas de mensajes y otras comunicaciones inter-servicio.

El mecanismo subyacente es el bytecode instrumentation en el caso de JVM (usando Java Agents y la API de instrumentacion) o monkey patching en lenguajes como Python o Node.js. La libreria de instrumentacion se carga junto con la aplicacion y redefine en runtime los metodos de las bibliotecas HTTP, las drivers de bases de datos y los clientes de colas para que, antes y despues de cada operacion, se cree y cierre un span automaticamente.

La ventaja es la facilidad de implementacion: se anade una dependencia, se configura el endpoint del collector y la aplicacion empieza a emitir trazas sin modificaciones de codigo. Esto la convierte en el punto de partida ideal para adoptar tracing en organizaciones grandes.

El compromiso es que solo captura operaciones de comunicacion inter-servicio. La instrumentacion automatica no sabe que una secuencia de queries SQL corresponde a la validacion de stock de un pedido. No puede anadir contexto de negocio a los spans. Ademas, la granularidad es limitada: una unica llamada HTTP externa se convierte en un span, pero no puedes subdividir internamente el procesamiento de la respuesta en sub-operaciones significativas.

> La instrumentacion automatica y la manual no son excluyentes. La practica recomendada es combinar ambas: instrumentacion automatica para las comunicaciones inter-servicio (que cubren la mayoria de las necesidades de tracing distribuido) y instrumentacion manual para las operaciones de negocio criticas donde se necesita granularidad adicional.

## OpenTelemetry

OpenTelemetry (OTel) surgió como respuesta a un problema fragmentado del ecosistema de observabilidad. Antes de su existencia, cada plataforma de observabilidad (Datadog, Dynatrace, New Relic, Jaeger, Zipkin) proponia sus propias bibliotecas de instrumentacion, formatos de propagacion y protocolos de recoleccion. Instrumentar un servicio para multiples backends significaba anadir multiples SDKs incompatibles, cada uno con su overhead y su modelo de datos. Los desarrolladores enfrentaban un dilema de vendor lock-in desde el primer dia de instrumentacion.

OpenTelemetry se estableció como proyecto de la Cloud Native Computing Foundation (CNCF) con el objetivo de ser el estandar unificado para la telemetria: metricas, logs y traces. Su filosofia fundamental es la separacion entre la instrumentacion y el backend de observabilidad. Las aplicaciones instrumentan con el SDK de OpenTelemetry y exportan telemetria en formato generico. Un collector intermediario puede recibir esa telemetria, transformarla y reenviarla a cualquier numero de backends sin modificar la aplicacion.

Esta arquitectura ofrece tres beneficios fundamentales. Primero, vendor neutrality: si mañana decides cambiar de Datadog a Grafana Cloud, solo cambias la configuracion del collector, no el codigo de tus aplicaciones. Segundo, un ecosistema de instrumentacion unificado: una sola biblioteca de instrumentacion automatica cubre multiples frameworks y librerias, sin necesidad de mantener SDKs separados por backend. Tercero, la comunidad es el motor del ecosistema: como proyecto CNCF, OpenTelemetry tiene contribuciones de todas las principales empresas de tecnologia y plataformas de observabilidad, garantizando soporte amplio y actualizaciones continuas.

```text
┌─────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Aplicacion  │────>│ OTel Collector  │────>│ Backend A (SaaS) │
│  (SDK OTel)  │     │                 │────>│ Backend B (local)│
└─────────────┘     └─────────────────┘     └──────────────────┘
```

El SDK de OpenTelemetry esta disponible en multiples lenguajes (Java, Python, Go, JavaScript, .NET, Rust, Ruby, PHP, Erlang y mas) con soporte tanto para instrumentacion automatica como manual. La instrumentacion automatica se anade tipicamente como una dependencia de la aplicacion y configura los propagadores y exporters necesarios. La instrumentacion manual se realiza usando la API de OpenTelemetry directamente, creando spans y anadiendo atributos donde se necesite granularidad adicional.

> OpenTelemetry se ha convertido en el segundo proyecto de CNCF con mas contribuciones despues de Kubernetes. Su rapida adopcion no es accidental: resuelve un problema real de fragmentacion que afectaba a toda la industria. Hoy, practicamente todas las plataformas de observabilidad soportan ingesta de datos de OpenTelemetry, lo que convierte a la migracion de OTel en una decision de bajo riesgo.

