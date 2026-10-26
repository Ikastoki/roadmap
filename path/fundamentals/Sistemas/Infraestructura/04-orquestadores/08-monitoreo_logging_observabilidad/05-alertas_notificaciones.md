# Alertas y notificaciones

Las metricas, los logs y las trazas solo son utiles si alguien las revisa o si existen mecanismos automaticos que reaccionen ante condiciones anomalas. Las alertas son el vinculo entre la observabilidad pasiva (dashboards que alguien consulta) y la accion (notificaciones que llegan a quien debe intervenir). Un sistema de alertas bien configurado reduce el tiempo medio de deteccion (MTTD) y el tiempo medio de resolucion (MTTR) de las incidencias, que son las dos metricas que definen la efectividad operativa de un equipo de plataforma.

El proceso de alertado sigue un flujo definido: una condicion se evalua periodicamente contra los datos de metricas, y cuando se cumple durante un tiempo suficiente, se genera una alerta que se envia a un receptor (persona o sistema). La clave esta en los dos primeros pasos: definir la condicion correcta y evitar la fatiga de alertas. Un sistema que genera demasiadas alertas falsas (false positives) termina siendo ignorado por los equipos; uno que genera demasiadas pocas deja pasar problemas reales.

> El concepto de "alert fatigue" es uno de los problemas mas serios en operaciones. Un estudio de Gartner estima que el 25-50% de las alertas en entornos de TI son falsos positivos. El objetivo no es eliminar las alertas, sino que cada alerta que llegue a un ingeniero represente una accion real que necesita atencion.

## Definicion de condiciones

En Prometheus, las alertas se definen como reglas de tipo `alerting` dentro de un `PrometheusRule`. Cada regla contiene una expresion PromQL que se evalua periodicamente y un conjunto de condiciones que determinan cuando la alerta se activa y cuando se desactiva.

```yaml
# PrometheusRule con alertas para el clúster
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: alertas-clUSTER
  namespace: monitoreo
  labels:
    release: prometheus
spec:
  groups:
    - name: cluster-alerts
      rules:
        # Alerta: Pod reiniciandose frecuentemente
        - alert: PodReiniciosFrecuentes
          expr: |
            increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 5m
          labels:
            severity: warning
            team: plataforma
          annotations:
            summary: "Pod {{ $labels.pod }} se ha reiniciado {{ $value }} veces en la ultima hora"
            description: >
              El pod {{ $labels.pod }} en el namespace {{ $labels.namespace }}
              ha experimentado mas de 5 reinicios en la ultima hora.
              Esto puede indicar un problema de OOM, health check fallido
              o un bug en la aplicacion.

        # Alerta: Uso de memoria criticamente alto
        - alert: MemoriaCritica
          expr: |
            (
              container_memory_working_set_bytes{namespace!=""}
              /
              kube_pod_container_resource_limits{resource="memory", namespace!=""}
            ) > 0.95
          for: 10m
          labels:
            severity: critical
            team: plataforma
          annotations:
            summary: "Pod {{ $labels.pod }} usa mas del 95% de su limite de memoria"
            description: >
              El pod {{ $labels.pod }} esta usando mas del 95% del limite de memoria
              asignado durante 10 minutos consecutivos. Si no se libera memoria,
              el contenedor sera terminado por el kernel OOM.

        # Alerta: Nodo en estado NotReady
        - alert: NodoNotReady
          expr: |
            kube_node_status_condition{condition="Ready", status="true"} == 0
          for: 2m
          labels:
            severity: critical
            team: plataforma
          annotations:
            summary: "Nodo {{ $labels.node }} en estado NotReady"
            description: >
              El nodo {{ $labels.node }} lleva mas de 2 minutos en estado NotReady.
              Los pods en este nodo pueden ser reprogramados a otros nodos
              si existen tolerancias y afinidad adecuadas.

        # Alerta: Latencia alta en la API server
        - alert: APIServerLatenciaAlta
          expr: |
            histogram_quantile(0.99,
              sum(rate(apiserver_request_duration_seconds_bucket[5m]))
              by (verb, le)
            ) > 1
          for: 5m
          labels:
            severity: warning
            team: plataforma
          annotations:
            summary: "API server con latencia P99 superior a 1 segundo"
            description: >
              La latencia del percentil 99 de la API server para el verbo
              {{ $labels.verb }} supera 1 segundo desde hace mas de 5 minutos.
              Esto afecta a toda la orquestacion del clúster.

        # Alerta: Disco casi lleno en un nodo
        - alert: DiscoCritico
          expr: |
            (
              node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
              /
              node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}
            ) < 0.1
          for: 5m
          labels:
            severity: critical
            team: plataforma
          annotations:
            summary: "Nodo {{ $labels.instance }} con menos del 10% de disco libre"
            description: >
              El nodo {{ $labels.instance }} tiene menos del 10% de espacio
              disponible en el volumen {{ $labels.mountpoint }}.
              Los logs y las imagenes de contenedores pueden agotar el espacio
              y causar fallos en los pods.

    - name: application-alerts
      rules:
        # Alerta: Tasa de errores HTTP elevada
        - alert: TasaErroresHTTP
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (app)
            /
            sum(rate(http_requests_total[5m])) by (app)
            > 0.05
          for: 5m
          labels:
            severity: warning
            team: desarrollo
          annotations:
            summary: "Servicio {{ $labels.app }} con mas del 5% de errores HTTP 5xx"
            description: >
              El servicio {{ $labels.app }} esta devolviendo mas del 5% de
              respuestas con error de servidor (5xx) durante los ultimos 5 minutos.

        # Alerta: Peticiones latencia alta
        - alert: LatenciaP99Alta
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (app, endpoint)
            ) > 2
          for: 5m
          labels:
            severity: warning
            team: desarrollo
          annotations:
            summary: "Endpoint {{ $labels.endpoint }} en {{ $labels.app }} con P99 > 2s"
            description: >
              El endpoint {{ $labels.endpoint }} del servicio {{ $labels.app }}
              tiene una latencia del percentil 99 superior a 2 segundos.
```

## Alertmanager: enrutamiento y notificacion

Prometheus genera las alertas, pero es Alertmanager quien gestiona el enrutamiento, la agrupacion, la inhibicion y el envio a los sistemas de notificacion. Alertmanager se despliega como un componente independiente y Prometheus le envia las alertas activas a traves de su API.

```yaml
# Configuracion de Alertmanager
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoreo
data:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.empresa.com:587'
      smtp_from: 'alertas@empresa.com'
      smtp_auth_username: 'alertas@empresa.com'
      smtp_auth_password: '${SMTP_PASSWORD}'

    # Plantillas de notificacion
    templates:
      - '/etc/alertmanager/templates/*.tmpl'

    # Regla de agrupacion: agrupa alertas similares en una unica notificacion
    # durante 5 minutos antes de enviar
    route:
      receiver: 'equipo-plataforma-default'
      group_by: ['alertname', 'namespace', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        # Alertas criticas van directo a PagerDuty
        - match:
            severity: critical
          receiver: 'pagerduty-criticas'
          group_wait: 10s
          repeat_interval: 1h

        # Alertas de desarrollo van a Slack
        - match:
            team: desarrollo
          receiver: 'slack-desarrollo'
          group_by: ['alertname', 'app']

        # Alertas de plataforma van a Slack
        - match:
            team: plataforma
          receiver: 'slack-plataforma'
          group_by: ['alertname', 'namespace']

    # Inhibiciones: si hay una alerta critica, suprimir las warning
    # del mismo componente
    inhibit_rules:
      - source_match:
          severity: critical
        target_match:
          severity: warning
        equal: ['namespace', 'alertname']

    # Receptores de notificacion
    receivers:
      - name: 'equipo-plataforma-default'
        email_configs:
          - to: 'plataforma@empresa.com'

      - name: 'pagerduty-criticas'
        pagerduty_configs:
          - routing_key: '${PAGERDUTY_KEY}'
            description: '{{ .GroupLabels.alertname }} - {{ .GroupLabels.namespace }}'
            severity: '{{ if eq .GroupLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
            details:
              firing: '{{ .Alerts.Firing | len }} alertas activas'
              resolved: '{{ .Alerts.Resolved | len }} alertas resueltas'

      - name: 'slack-desarrollo'
        slack_configs:
          - api_url: '${SLACK_WEBHOOK_URL}'
            channel: '#alerts-desarrollo'
            title: '[{{ .GroupLabels.severity | toUpper }}] {{ .GroupLabels.alertname }}'
            text: >-
              {{ range .Alerts }}
              *Alerta:* {{ .Annotations.summary }}
              *Descripcion:* {{ .Annotations.description }}
              *Namespace:* {{ .Labels.namespace }}
              *Estado:* {{ .Status }}
              {{ end }}
            send_resolved: true

      - name: 'slack-plataforma'
        slack_configs:
          - api_url: '${SLACK_WEBHOOK_URL}'
            channel: '#alerts-plataforma'
            title: '[{{ .GroupLabels.severity | toUpper }}] {{ .GroupLabels.alertname }}'
            text: >-
              {{ range .Alerts }}
              *Alerta:* {{ .Annotations.summary }}
              *Descripcion:* {{ .Annotations.description }}
              *Estado:* {{ .Status }}
              {{ end }}
            send_resolved: true
```

## Escalacion por severidad

La escalacion de alertas es un proceso temporal que garantiza que las alertas no criticas se resuelven antes de escalar a mas personas o sistemas. Un esquema tipico de escalacion en tres niveles es:

| Nivel | Tiempo | Accion | Canal |
|-------|--------|--------|-------|
| 1 | 0-15 min | Notificacion al equipo responsable | Slack, email |
| 2 | 15-30 min | Escalacion al lead de equipo | PagerDuty, SMS |
| 3 | 30+ min | Escalacion a on-call manager | Telefono, page |

```
# Flujo de escalacion con Alertmanager
#
# Alerta generada
#     │
#     ▼
# group_wait: 30s (agrupar alertas similares)
#     │
#     ▼
# route por severity
#     │
#     ├── severity=warning → Slack equipo (repeat: 4h)
#     │
#     └── severity=critical → PagerDuty (repeat: 1h)
#              │
#              ▼ (si no se reconoce en 15 min)
#         Escalacion a manager
#              │
#              ▼ (si no se resuelve en 30 min)
#         Escalacion a directivo
```

> La configuracion de `repeat_interval` determina cada cuanto tiempo se reenvia una alerta que no se ha resuelto ni reconocido. En entornos criticos, un repeat interval de 1 hora para alertas criticas asegura que el problema no quede olvidado. Para alertas warning, 4 horas es un valor razonable que evita la fatiga sin perder visibilidad.

## Mute rules y silenciamiento

Existen situaciones donde las alertas conocidas deben ser silenciadas temporalmente: mantenimientos planificados, despliegues controlados o ventanas de actualizacion. Alertmanager proporciona dos mecanismos para esto:

- **Silencimiento (silences)**: suprime temporalmente todas las alertas que coincidan con un filtro durante un periodo definido. Se configuran manualmente o a traves de la API.
- **Mute rules**: reglas de supresion basadas en condiciones de las alertas (por ejemplo, suprimir todas las alertas de un namespace durante un maintenance window).

```yaml
# Ejemplo de silencimiento via API de Alertmanager
# POST /api/v2/silences
{
  "matchers": [
    {
      "name": "namespace",
      "value": "mantenimiento",
      "isRegex": false
    }
  ],
  "startsAt": "2026-07-27T02:00:00Z",
  "endsAt": "2026-07-27T06:00:00Z",
  "createdBy": "operador-plataforma",
  "comment": "Mantenimiento programado del cluster - actualizacion de nodos",
  "createdBy": "ci-pipeline"
}
```

