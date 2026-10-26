# Trazas distribuidas

En un sistema monolitico, una peticion del cliente se resuelve dentro de un unico proceso. El stack de llamadas es lineal y accesible desde un unico punto de depuracion. En un sistema de microservicios, esa misma peticion puede atravesar diez o mas servicios, cada uno con su propio ciclo de vida, su propio log y su propia metrica. Sin trazas distribuidas, reconstruir el camino que siguio una peticion desde el API gateway hasta la base de datos requiere cruzar manualmente logs de multiples servicios, un proceso que en sistemas con trafico real es practicamente imposible.

Una traza distribuida captura la secuencia completa de operaciones que se ejecutan para procesar una unica peticion, independientemente de cuantos servicios participen. Cada operacion individual se registra como un span, y todos los spans que pertenecen a la misma peticion se agrupan bajo un trace ID unico. Esta estructura jerarquica permite no solo ver el camino completo de la peticion, sino tambien identificar exactamente en que punto se invierte el tiempo y donde ocurren los fallos.

## Conceptos fundamentales

Los conceptos centrales de las trazas distribuidas se derivan del modelo de OpenTelemetry y son compartidos por todas las herramientas de la ecosistema (Jaeger, Zipkin, etc.):

**Trace**: identificador unico que representa el recorrido completo de una peticion a traves de todos los servicios. Se genera en el punto de entrada del sistema (API gateway, balanceador de carga) y se propaga a traves de headers HTTP.

**Span**: una unidad de trabajo dentro de una traza. Cada span contiene: un nombre descriptivo, una marca de tiempo de inicio y fin, un trace ID comun, un parent span ID (que establece la jerarquia) y un conjunto de atributos (key-value) con metadatos del contexto.

**Context propagation**: el mecanismo por el cual el trace ID y el parent span ID se transmiten de un servicio a otro. En HTTP, esto se hace tipicamente mediante headers `traceparent` (estandar W3C) o `uber-trace-id` (formato Jaeger).

```
# Header W3C Trace Context (estandar)
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

# Estructura: version-trace_id-parent_id-trace_flags
# version:     00
# trace_id:    4bf92f3577b34da6a3ce929d0e0e4736 (128 bits)
# parent_id:   00f067aa0ba902b7 (64 bits)
# trace_flags: 01 (sampled)
```

## Jaeger

Jaeger es un sistema de trazas distribuidas open source, originado en Uber y actualmente un proyecto CNCF. Sigue la arquitectura de agent-collector-query con almacenamiento backend. El agent se despliega como sidecar o DaemonSet y recibe spans de las aplicaciones. El collector procesa y almacena los spans en el backend. El servicio de query expone una interfaz web y una API REST para consultar las trazas.

```yaml
# Despliegue de Jaeger en modo production con Elasticsearch
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-principal
  namespace: observabilidad
spec:
  strategy: production
  collector:
    replicas: 3
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        memory: 1Gi
    options:
      es:
        server-urls: https://elasticsearch.logging.svc:9200
        index-prefix: jaeger
        num-shards: 3
        num-replicas: 1
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch.logging.svc:9200
  query:
    replicas: 2
    options:
      query-base-path: /jaeger
  agent:
    strategy: DaemonSet
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        memory: 256Mi
```

### Instrumentacion de una aplicacion con Jaeger

```python
# Ejemplo de instrumentacion en Python usando opentelemetry
# y el exporter de Jaeger
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from fastapi import FastAPI

# Configurar el recurso (identifica el servicio)
resource = Resource.create({
    "service.name": "servicio-pedidos",
    "service.version": "1.2.0",
    "deployment.environment": "produccion"
})

# Configurar el provider de trazas
provider = TracerProvider(resource=resource)

# Configurar el exporter hacia Jaeger
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent.observabilidad",
    agent_port=6831,
)
provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))

# Establecer el provider global
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("servicio-pedidos.tracer")

# Instrumentar FastAPI automaticamente
app = FastAPI()
FastAPIInstrumentor.instrument_app(app)

# Uso manual de trazas en el codigo de la aplicacion
@app.post("/api/v1/pedidos")
async def crear_pedido(pedido: dict):
    with tracer.start_as_current_span("validar-stock") as span:
        span.set_attribute("pedido.items", len(pedido["items"]))
        resultado = await validar_stock(pedido["items"])

        if not resultado.disponible:
            span.set_attribute("pedido.stock_suficiente", False)
            span.add_event("stock_insuficiente", {
                "items_faltantes": str(resultado.faltantes)
            })
            return {"error": "Stock insuficiente"}, 409

        span.set_attribute("pedido.stock_suficiente", True)

    with tracer.start_as_current_span("persistir-pedido") as span:
        pedido_id = await guardar_pedido(pedido)
        span.set_attribute("pedido.id", pedido_id)
        return {"pedido_id": pedido_id}, 201
```

## Zipkin

Zipkin es otro sistema de trazas distribuidas ampliamente utilizado, con una interfaz web intuitiva para visualizar las trazas. Su arquitectura es similar a Jaeger pero con una comunidad mas orientada a entornos JVM. La seleccion entre Jaeger y Zipkin en la practica depende del ecosistema existente: Jaeger es la eleccion preferida en entornos Kubernetes nativos y con OpenTelemetry, mientras Zipkin sigue siendo popular en ecosistemas Spring Boot.

```yaml
# Despliegude basico de Zipkin
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zipkin
  namespace: observabilidad
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zipkin
  template:
    metadata:
      labels:
        app: zipkin
    spec:
      containers:
        - name: zipkin
          image: openzipkin/zipkin:3
          ports:
            - containerPort: 9411
              name: http
          env:
            - name: STORAGE_TYPE
              value: elasticsearch
            - name: ES_HOSTS
              value: "http://elasticsearch.logging.svc:9200"
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              memory: 512Mi
```

## Analisis de trazas

La verdadera utilidad de las trazas distribuidas se manifiesta en el analisis. Jaeger y Zipkin proporcionan una interfaz grafica donde se pueden buscar trazas por servicio, operacion, duracion, atributos custom y estado (error/success). El flujo de diagnostico tipico es:

1. Filtrar trazas por servicio afectado y rango temporal.
2. Ordenar por duracion (las mas lentas primero).
3. Inspeccionar el waterfall diagram para identificar el span que introdujo la mayor latencia.
4. Revisar los atributos y eventos del span problematico.
5. Comparar con trazas exitosas equivalentes para identificar la divergencia.

```
# Ejemplo de waterfall diagram en Jaeger para una peticion lenta

Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
Duracion total: 2.340ms

[0ms ─────────────────────────────────── 2340ms] api-gateway (2340ms)
  [5ms ──── 25ms] authenticator (20ms)
  [30ms ──────────────────────────── 2310ms] servicio-pedidos (2280ms)
    [35ms ─── 55ms] validar-stock (20ms)
    [60ms ────────────────────────── 2300ms] persistir-pedido (2240ms) ⚠️
      [65ms ─── 80ms] postgres-consultar (15ms)
      [85ms ────────────────────────── 2290ms] postgres-insertar (2205ms) ⚠️
        [90ms ─── 110ms] redis-cache (20ms)
        [115ms ──────────────────────── 2285ms] postgres-write-lock (2170ms) ⚠️
```

En este ejemplo, la traza revela que el cuello de botella es un lock de escritura en PostgreSQL que se mantiene durante 2170ms, lo que retrasa toda la cadena. Sin la traza distribuida, este problema solo se manifestaria como una latencia alta en el API gateway, sin poder atribuirla a un componente especifico.

## OpenTelemetry como capa unificada

OpenTelemetry (OTel) ha emergido como el estandar de facto para la instrumentacion de trazas, metricas y logs. A diferencia de Jaeger o Zipkin, que son backend de trazas, OTel es una capa de instrumentacion y recoleccion que se conecta con cualquier backend. Esto permite cambiar de Jaeger a Zipkin o a cualquier otro sistema sin modificar el codigo de la aplicacion.

```yaml
# OTel Collector configurado para trazas
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-traces
  namespace: observabilidad
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 2s
        send_batch_size: 8192
      memory_limiter:
        check_interval: 1s
        limit_mib: 256

    exporters:
      otlp/jaeger:
        endpoint: jaeger-collector.observabilidad:4317
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/jaeger]
```

> El muestreo (sampling) es fundamental en produccion. Enviar el 100% de las trazas a un sistema de almacenamiento es inviable con trafico real. El muestreo probabilistico (por ejemplo, el 1% de las trazas) o el muestreo adaptativo (100% de errores, 1% de exitosos) permite mantener el coste de almacenamiento bajo mientras se preservan las trazas mas relevantes para diagnostico.

