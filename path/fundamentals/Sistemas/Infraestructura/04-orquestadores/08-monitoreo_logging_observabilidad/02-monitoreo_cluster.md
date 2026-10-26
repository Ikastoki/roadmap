# Monitoreo del cluster

El monitoreo de un clúster Kubernetes se estructura en tres capas jerárquicas: el plano de control, los nodos y las unidades de trabajo (pods). Cada capa expone metricas distintas que, combinadas, permiten determinar si el clúster esta saludable, si los recursos son suficientes y si las aplicaciones se comportan correctamente. Un fallo en cualquiera de estas capas tiene consecuencias distintas: un etcd lento degrada toda la orquestación, un nodo sin memoria afecta a los pods que aloja y un pod con reinicios frecuentes indica un problema en la aplicacion.

El plano de control es el cerebro del clúster y su monitoreo es no negociable. La API server, etcd, el scheduler y el controller manager exponen metricas de rendimiento propias que reflejan la salud global del sistema. Si la latencia de la API server supera los 500ms, toda operacion de orquestacion (programacion de pods, reconciliacion de controladores) se ralentiza, incluso si las aplicaciones estan sanas. Monitorear etcd es especialmente critico porque es el componente unico (single point of failure en terminos de disponibilidad) cuya degradacion puede hacer que el clúster pierda capacidad de toma de decisiones.

## Metricas del plano de control

Cada componente del plano de control expone un endpoint `/metrics` en formato Prometheus. Las metricas mas relevantes son:

```yaml
# Endpoints de metricas del plano de control
# API Server (por defecto en el puerto 6443)
# https://<api-server>:6443/metrics

# Etcd (puerto 2379, endpoint /metrics)
# https://<etcd>:2379/metrics

# Scheduler (puerto 10259)
# https://<scheduler>:10259/metrics

# Controller Manager (puerto 10257)
# https://<controller-manager>:10257/metrics
```

Las metricas criticas del API server incluyen:

| Metrica | Descripcion | Umbral recomendado |
|---------|-------------|-------------------|
| `apiserver_request_duration_seconds` | Latencia de cada tipo de operacion | P99 < 1s |
| `apiserver_request_total` | Volumen de peticiones por verb y recurso | Estable, sin picos |
| `apiserver_current_inflight_requests` | Peticiones concurrentes en proceso | < 400 |
| `etcd_request_duration_seconds` | Tiempo de operacion en etcd | P99 < 100ms |

```yaml
# ServiceMonitor para el API server y etcd
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: control-plane-metrics
  namespace: monitoreo
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      component: kube-apiserver
  endpoints:
    - port: https-metrics
      scheme: HTTPS
      tlsConfig:
        caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecureSkipVerify: true
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      path: /metrics
      interval: 30s
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-metrics
  namespace: monitoreo
spec:
  selector:
    matchLabels:
      k8s-app: etcd
  endpoints:
    - port: etcd-metrics
      scheme: HTTPS
      tlsConfig:
        insecureSkipVerify: true
      path: /metrics
      interval: 15s
```

## Metricas de nodos

El kubelet en cada nodo expone metricas del sistema operativo y del estado de los pods que aloja. Ademas, los nodos deben monitorizarse por su capacidad de recursos y por eventos que indiquen problemas de infraestructura: presion de memoria, presion de disco, problemas de red o fallos en los dispositivos de almacenamiento.

```yaml
# ServiceMonitor para metricas de nodos a traves de kubelet
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubelet-metrics
  namespace: monitoreo
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      k8s-app: kubelet
  endpoints:
    - port: http-metrics
      path: /metrics
      interval: 30s
    - port: http-metrics
      path: /metrics/cadvisor
      interval: 30s
    - port: http-metrics
      path: /metrics/probes
      interval: 30s
```

Las metricas de nodos mas relevantes son:

- `node_cpu_seconds_total`: tiempo de CPU en cada modo (idle, system, user). Se usa para calcular el porcentaje de uso real.
- `node_memory_MemAvailable_bytes`: memoria disponible, incluyendo buffers y cache reutilizable.
- `node_filesystem_avail_bytes`: espacio disponible en disco para los volumenes relevantes.
- `node_network_receive_bytes_total`: trafico de red entrante, util para detectar picos anomales.
- `kubelet_running_pods`: numero de pods activos en el nodo.
- `kubelet_pleg_relist_duration_seconds`: latencia del Pod Lifecycle Event Generator, un indicador sensible de problemas en el kubelet.

> Si `kubelet_pleg_relist_duration_seconds` supera los 10 segundos de forma sostenida, el nodo esta experimentando una degradacion severa y es candidato a ser marcado como `NotReady` por el controlador de nodos.

## Metricas de unidades de trabajo

Las metricas de pods y contenedores se obtienen a traves de cAdvisor (integrado en el kubelet) y del metrics-server. Estas metricas permiten identificar aplicaciones que consumen recursos excesivos, que se reinician frecuentemente o que no responden a los health checks configurados.

```yaml
# Ejemplo de Grafana dashboard JSON para monitoreo de pods
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-pods
  namespace: monitoreo
  labels:
    grafana_dashboard: "1"
data:
  pods-dashboard.json: |
    {
      "dashboard": {
        "title": "Monitoreo de Pods - Kubernetes",
        "uid": "k8s-pods-overview",
        "tags": ["kubernetes", "pods"],
        "panels": [
          {
            "title": "Uso de CPU por Pod",
            "type": "timeseries",
            "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
            "targets": [
              {
                "expr": "sum(rate(container_cpu_usage_seconds_total{namespace!=\"\"}[5m])) by (pod)",
                "legendFormat": "{{pod}}",
                "refId": "A"
              }
            ],
            "fieldConfig": {
              "defaults": {
                "unit": "percentunit",
                "custom": {
                  "drawStyle": "line",
                  "lineWidth": 1,
                  "fillOpacity": 20
                }
              }
            }
          },
          {
            "title": "Memoria por Pod",
            "type": "timeseries",
            "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
            "targets": [
              {
                "expr": "sum(container_memory_working_set_bytes{namespace!=\"\"}) by (pod)",
                "legendFormat": "{{pod}}",
                "refId": "A"
              }
            ],
            "fieldConfig": {
              "defaults": {
                "unit": "bytes",
                "custom": {
                  "drawStyle": "line",
                  "lineWidth": 1,
                  "fillOpacity": 20
                }
              }
            }
          },
          {
            "title": "Reinicios de Contenedores",
            "type": "stat",
            "gridPos": { "h": 4, "w": 8, "x": 0, "y": 8 },
            "targets": [
              {
                "expr": "increase(kube_pod_container_status_restarts_total[1h]) > 0",
                "legendFormat": "{{pod}}",
                "refId": "A"
              }
            ],
            "fieldConfig": {
              "defaults": {
                "thresholds": {
                  "steps": [
                    { "color": "green", "value": 0 },
                    { "color": "yellow", "value": 1 },
                    { "color": "red", "value": 5 }
                  ]
                }
              }
            }
          },
          {
            "title": "Estado de Pods",
            "type": "table",
            "gridPos": { "h": 6, "w": 16, "x": 8, "y": 8 },
            "targets": [
              {
                "expr": "kube_pod_status_phase",
                "legendFormat": "{{pod}}",
                "refId": "A",
                "format": "table",
                "instant": true
              }
            ]
          }
        ],
        "time": { "from": "now-1h", "to": "now" },
        "refresh": "30s"
      }
    }
```

## Herramientas de monitoreo

### Prometheus y Grafana

Prometheus es el sistema de recoleccion y almacenamiento de metricas por defecto en ecosistemas Kubernetes. Gracias al Operator de Prometheus, su despliegue y configuracion se gestiona mediante recursos custom (CRDs) como `Prometheus`, `ServiceMonitor` y `PrometheusRule`. Grafana se conecta a Prometheus como fuente de datos y permite construir dashboards visualmente.

```yaml
# Despliegue basico del Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-principal
  namespace: monitoreo
spec:
  replicas: 2
  retention: 15d
  resources:
    requests:
      memory: 2Gi
      cpu: 500m
    limits:
      memory: 4Gi
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
  serviceMonitorSelector:
    matchLabels:
      release: prometheus
  serviceAccountName: prometheus
```

### Datadog y New Relic

Las plataformas SaaS como Datadog y New Relic ofrecen monitoreo completo (metricas, logs y trazas) como servicio gestionado. Se despliegan mediante un DaemonSet que actua como agente, recolectando metricas del nodo, de Kubernetes y de las aplicaciones. La ventaja principal es la ausencia de infraestructura de monitoreo que mantener; la desventja es el coste variable por host o por volumen de datosinger.

> Prometheus con Grafana es la combinacion mas utilizada en entornos Kubernetes auto-gestionados. Datadog y New Relic son preferidos cuando el equipo no quiere operar la infraestructura de monitoreo o cuando se necesita una plataforma unificada que incluya APM, log management y monitoreo de infraestructura en un solo producto.

