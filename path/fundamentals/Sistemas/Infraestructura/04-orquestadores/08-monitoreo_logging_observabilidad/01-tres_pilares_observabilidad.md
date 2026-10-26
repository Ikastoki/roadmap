# Los tres pilares de la observabilidad

La observabilidad en sistemas distribuidos es la capacidad de inferir el estado interno de un clúster y sus aplicaciones a partir de las respuestas que genera frente a estímulos externos. A diferencia del monitoreo tradicional, que responde a preguntas conocidas ("¿el CPU supera el 80%?"), la observabilidad permite formular preguntas nuevas sobre comportamientos que no se anticiparon en el diseño del sistema. En un entorno Kubernetes donde decenas de pods se comunican entre sí a través de servicios de red, la combinación de métricas, logs y trazas distribuidas constituye la única forma de obtener una visión coherente y completa de lo que ocurre en cada nivel de la pila.

Los tres pilares no compiten entre sí; cada uno responde a una categoría distinta de pregunta. Las métricas dicen **cuánto** y **cuándo**, los logs dicen **qué ocurrió** y las trazas dicen **dónde se invirtió el tiempo**. Un error en una aplicación se detecta primero en las métricas (tasa de errores 5xx), se investiga en los logs (mensaje de excepción) y se contextualiza en las trazas (cuál de los cinco servicios implicados introdujo la latencia). Negligir uno de los pilares crea puntos ciegos que en entornos de alta disponibilidad se traducen en incidencias no detectadas o diagnosticadas incorrectamente.

> La regla práctica es que las métricas te alertan, los logs te explican y las trazas te sitúan. Si solo tienes métricas, sabes que algo falla pero no por qué. Si solo tienes logs, tienes mucha información pero no la estructura temporal de una petición concreta. Si solo tienes trazas, ves un camino de llamadas pero no el volumen agregado del problema.

## Metricas

Las metricas son valores numericos agregados en el tiempo, almacenados en series temporales. Cada serie se identifica por un nombre (como `cpu_usage_percent`) y un conjunto de etiquetas (como `namespace="produccion"`, `pod="api-7f3b2"`). Esta estructura permite realizar agregaciones sobre dimensiones arbitrarias: calcular el uso medio de memoria por nodo, el percentil 99 de latencia por endpoint o la tasa de reinicios de pods por namespace.

En Kubernetes, las metricas se obtienen de dos fuentes principales:

- **Metricas del nodo y del clúster**: recolectadas por `kubelet` y expuestas a través de la API de metricas (`metrics-server`) o del endpoint `/metrics` de cada componente del plano de control.
- **Metricas de la aplicacion**: expuestas por la propia aplicación usando un cliente de metricas (Prometheus client library) en un endpoint HTTP, normalmente `/metrics`.

```yaml
# Ejemplo de metrica expuesta por una aplicacion en formato Prometheus
# GET /metrics → response
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", endpoint="/api/v1/users", status="200"} 15234
http_requests_total{method="POST", endpoint="/api/v1/users", status="201"} 892
http_requests_total{method="GET", endpoint="/api/v1/users", status="500"} 47
```

Los tipos de metricas fundamentales en el ecosistema Prometheus son:

| Tipo | Descripcion | Ejemplo |
|------|-------------|---------|
| Counter | Valor acumulativo, solo sube | `http_requests_total` |
| Gauge | Valor que sube y baja | `memory_usage_bytes` |
| Histogram | Distribucion de valores en buckets | `request_duration_seconds` |
| Summary | Estadisticas calculadas en el cliente | `request_duration_seconds` |

## Logs

Los logs son registros discretos de eventos, cada uno con una marca temporal y un mensaje descriptivo. En Kubernetes, los contenedores escriben su salida estandar (`stdout`) y de error (`stderr`), que el runtime del contenedor (containerd, CRI-O) captura y almacena en el nodo. El daemon `kubelet` gestiona la rotacion de estos archivos bajo `/var/log/containers/` o `/var/log/pods/`.

Cada entrada de log contiene metadatos que la vinculan al contexto del clúster: el nombre del pod, el namespace, el nombre del contenedor y el identificador del nodo. Estos metadatos son esenciales para correlacionar logs de distintos componentes cuando se realizan busquedas centralizadas.

```bash
# Ver los logs de un pod directamente desde kubectl
kubectl logs -n produccion mi-app-7f3b2x9k4m

# Ver los logs de un contenedor especifico en un pod multi-contenedor
kubectl logs -n produccion mi-app-7f3b2x9k4m -c sidecar

# Ver logs anteriores de un contenedor que se reinicio
kubectl logs -n produccion mi-app-7f3b2x9k4m --previous

# Seguir los logs en tiempo real
kubectl logs -f -n produccion mi-app-7f3b2x9k4m
```

> En entornos de produccion, los logs nunca se consultan directamente desde los nodos. El principio fundamental es que los contenedores no deben almacenar logs de forma persistente: se escriben a stdout/stderr, un agente los recolecta y los envia a un sistema centralizado. Al reiniciar un contenedor o eliminar un pod, los logs locales se pierden, y eso es intencional.

## Trazas distribuidas

Una traza registra el recorrido completo de una peticion a traves de multiples servicios. Cada traza esta compuesta por spans, donde cada span representa una operacion individual (una llamada HTTP, una consulta a base de datos, una operacion en cola de mensajes). Los spans se anidan o encadenan segun la relacion causal entre las operaciones.

En un sistema de microservicios real, una unica peticion del usuario puede atravesar un balanceador de carga, un API gateway, tres servicios de backend, una cache Redis y una base de datos PostgreSQL. Sin trazas distribuidas, identificar cual de estos componentes introduce 200ms de latencia adicional requiere revisar logs de cada servicio manualmente y reconstruir la cadena de llamadas. Con trazas distribuidas, esa informacion esta disponible en una sola visualizacion jerarquica.

Un trace tipico en formato simplificado se ve asi:

```
Trace: POST /api/v1/pedidos (total: 342ms)
├── Span: api-gateway → recepcion (2ms)
├── Span: servicio-pedidos → validar-stock (85ms)
│   ├── Span: postgres → SELECT inventory (80ms)
├── Span: servicio-pedidos → crear-pedido (95ms)
│   ├── Span: postgres → INSERT orders (12ms)
│   ├── Span: redis → SET cache_invalidation (3ms)
├── Span: servicio-pedidos → publicar-evento (120ms)
│   ├── Span: kafka → PRODUCE order.created (45ms)
```

## Complementariedad entre pilares

Los tres pilares forman un sistema de triangulacion. Ante una degradacion del servicio, el flujo de diagnostico natural es:

1. **Metricas** detectan la anomalia: la latencia P99 sube de 100ms a 2s.
2. **Trazas** identifican el componente: el span de `servicio-pagos` consume 1.8s, los demas estan dentro de lo normal.
3. **Logs** revelan la causa: en los logs de `servicio-pagos` aparecen multiples reintentos de conexion a la pasarela de pago.

Cada pilar sin los otros dos es insuficiente. Las metricas sin logs y trazas no dicen por que algo falla. Las logs sin metricas y trazas no muestran el patron temporal ni la propagacion del error. Las trazas sin metricas no muestran la frecuencia del problema a lo largo del tiempo.

## OpenTelemetry como estandar

OpenTelemetry es el proyecto CNCF que unifica metricas, logs y trazas bajo un unico conjunto de APIs y protocolos de recoleccion. Reemplaza a proyectos anteriores como OpenTracing y OpenCensus, y se ha convertido en el estandar de facto para la instrumentacion observabilidad en Kubernetes.

El agente de OpenTelemetry (OTel Collector) se despliega normalmente como DaemonSet o Deployment y actua como proxy intermedio: recibe datos instrumentados de las aplicaciones, los procesa (enriquece, filtra, transforma) y los exporta a los backend de observabilidad correspondientes.

```yaml
# Configuracion del OpenTelemetry Collector
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
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
      prometheus:
        config:
          scrape_configs:
            - job_name: 'kubernetes-pods'
              kubernetes_sd_configs:
                - role: pod

    processors:
      batch:
        timeout: 5s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 512
        spike_limit_mib: 128

    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"
      otlp/jaeger:
        endpoint: jaeger-collector.observabilidad:4317
        tls:
          insecure: true

    service:
      pipelines:
        metrics:
          receivers: [otlp, prometheus]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/jaeger]
```

> OpenTelemetry es solo la capa de recoleccion y transporte. No incluye almacenamiento ni visualizacion. Necesitas combinarlo con backends como Prometheus (metricas), Jaeger (trazas) y Loki (logs) para completar el stack de observabilidad.

