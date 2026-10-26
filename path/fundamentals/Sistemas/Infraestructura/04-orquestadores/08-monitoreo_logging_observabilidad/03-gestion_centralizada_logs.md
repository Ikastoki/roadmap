# Gestion centralizada de logs

En un clúster Kubernetes, los logs son efimeros por diseno. Cuando un contenedor escribe a stdout o stderr, el runtime del contenedor (containerd o CRI-O) captura esa salida y la almacena como un archivo en el nodo, tipicamente bajo `/var/log/containers/` o `/var/log/pods/`. Estos archivos tienen rotacion automatica y se eliminan cuando el pod termina o el nodo se reinicia. Esta politica es intencional: en un sistema orquestado donde los pods nacen y mueren continuamente, tratar de persistir logs localmente crea problemas de espacio en disco y dificulta la busqueda a traves de multiples nodos.

El flujo fundamental de gestion de logs en Kubernetes sigue una arquitectura de three-tier: los contenedores escriben a stdout/stderr, un agente DaemonSet recolecta esos logs en cada nodo y los envia a un sistema centralizado donde se almacenan, indexan y consultan. Esta separacion de responsabilidades permite escalar la busqueda y analisis de logs de forma independiente a la carga de las aplicaciones, y garantiza que los logs sobrevivan a la destruccion de pods y nodos.

> El error mas comun en entornos nuevos es intentar consultar logs con `kubectl logs` en produccion. Esto funciona en desarrollo, pero en un clúster con cientos de pods es inviable: no permite busquedas cruzadas, no persiste y no ofrece alertas basadas en patrones de logs. La regla es que `kubectl logs` es una herramienta de diagnostico inmediato, no de analisis.

## Pipeline de recoleccion

La recoleccion de logs en Kubernetes se basa en los logs de los contenedores almacenados por el runtime. Cada contenedor tiene un archivo de log en el nodo que contiene una linea por cada escritura a stdout o stderr, con metadatos附加 (nombre del pod, namespace, container ID) gestionados por el CRI log format.

```
# Estructura de un archivo de log en el nodo
# /var/log/containers/<pod_name>_<namespace>_<container_name>-<container_id>.log

# Formato de cada linea (CRI log format):
# <timestamp> <stream> <tag> <message>
2026-07-26T10:15:32.847Z stdout F {"level":"info","msg":"request received","method":"GET","path":"/api/v1/users"}
2026-07-26T10:15:32.901Z stderr F {"level":"error","msg":"database connection timeout","host":"postgres-svc"}
```

El agente de recoleccion se despliega como DaemonSet, lo que garantiza que exactamente un agente procese los logs de cada nodo. Los agentes mas utilizados son:

| Agente | Lenguaje | Ventaja principal |
|--------|----------|-------------------|
| Fluent Bit | C | Bajo consumo de recursos, ideal para nodos con restricciones |
| Fluentd | Ruby/C | Ecosistema amplio de plugins de entrada/salida |
| Promtail | Go | Integracion nativa con Loki |
| Logstash | Java | Flexibilidad de procesamiento, integracion con ELK |

## Fluent Bit como agente

Fluent Bit es la variante ligera de Fluentd, escrita en C con un consumo de memoria de 1-5MB. Es el agente recomendado para entornos donde el consumo de recursos del nodo es critico. Se despliega como DaemonSet y se configura para leer los archivos de log de los contenedores, enriquecerlos con metadatos de Kubernetes y enviarlos al backend de almacenamiento.

```yaml
# Fluent Bit DaemonSet para recoleccion de logs
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          ports:
            - containerPort: 2020
              name: http
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
          env:
            - name: FLUSH_INTERVAL
              value: "5"
            - name: FLUSH_COUNT
              value: "500"
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        multiline.parser  docker, cri
        Tag               kube.*
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        DB                /var/log/flb_kube.db

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name          es
        Match         kube.*
        Host          elasticsearch.logging.svc.cluster.local
        Port          9200
        Index         k8s-logs
        Type          _doc
        Logstash_Format On
        Logstash_Prefix k8s-logs
        Logstash_DateFormat %Y.%m.%d
  parsers.conf: |
    [PARSER]
        Name         docker
        Format       json
        Time_Key     time
        Time_Format  %Y-%m-%dT%H:%M:%S.%L%z

    [PARSER]
        Name         cri
        Format       regex
        Regex        ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key     time
        Time_Format  %Y-%m-%dT%H:%M:%S.%L%z
```

> Los tolerations en el DaemonSet son necesarios para que Fluent Bit se ejecute tambien en los nodos master/control-plane. Si los logs del plano de control estan en los mismos nodos que los workers, omitir estos tolerations deja ciega la mitad del cluster.

## Loki para almacenamiento y consulta

Grafana Loki es un sistema de almacenamiento de logs inspirado en Prometheus. A diferencia de Elasticsearch, que indexa el contenido completo de los logs, Loki indexa solo las etiquetas (labels), lo que reduce drasticamente el coste de almacenamiento. Los logs se almacenan en un formato similar a los chunks de Prometheus y se comprimen de forma eficiente.

La integracion de Loki con Grafana permite realizar busquedas de logs usando el mismo interfaz que las metricas, lo que facilita la correlacion entre anomalias detectadas en dashboards de metricas y los logs subyacentes.

```bash
# Consultas en LogQL (Loki Query Language)

# Buscar todos los logs del namespace produccion
{namespace="produccion"}

# Buscar logs de un pod especifico con un mensaje que contenga "error"
{namespace="produccion", pod="api-7f3b2"} |= "error"

# Buscar logs de stderr excluyendo un pod de health check
{namespace="produccion"} |= "" |~ "stderr" != "health-check"

# Contar errores por minuto en los ultimos 15 minutos
rate({namespace="produccion"} |= "error" [1m])

# Extraer un campo JSON de los logs y agruparlo
{namespace="produccion"} | json | line_format "{{.level}} {{.msg}}" | level="error"

# Buscar logs con patron regex
{app="servicio-pagos"} |~ "timeout|connection refused|ETIMEDOUT"

# Agregar metricas a partir de logs: tasa de errores por servicio
sum(rate({namespace="produccion"} |= "error" [5m])) by (app)
```

### Despliegue de Loki con Helm

```bash
# Instalacion de Loki con Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Instalar Loki (modo monolitico, ideal para clusters pequenos)
helm install loki grafana/loki \
  --namespace logging \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=50Gi

# Instalar Promtail (agente que envia logs a Loki)
helm install promtail grafana/promtail \
  --namespace logging \
  --set config.lokiAddress=http://loki.logging.svc:3100/loki/api/v1/push
```

## Correlacion entre metricas y logs

La verdadera potencia de la gestion centralizada de logs se alcanza cuando se correlacionan con las metricas. En Grafana, esto se logra mediante el panel "Explore" o configurando enlaces directos desde los dashboards de metricas hacia las consultas de logs correspondientes.

```yaml
# Ejemplo de dashboard de Grafana con links a Loki
# En la seccion "links" del dashboard JSON:
{
  "links": [
    {
      "title": "Ver logs en Loki",
      "type": "link",
      "url": "/explore?left={\"datasource\":\"Loki\",\"expr\":\"{namespace=\\\"$namespace\\\", pod=\\\"$pod\\\"}\"}",
      "icon": "doc"
    }
  ]
}
```

> La clave de una buena gestion de logs no es recolectar todo, sino poder encontrar lo relevante rapidamente. Estrategias como la estructuracion de logs en JSON, el uso consistente de niveles de log (info, warn, error) y la inclusion de identificadores de correlacion (request ID, trace ID) en cada mensaje facilitan enormemente el diagnostico en produccion.

