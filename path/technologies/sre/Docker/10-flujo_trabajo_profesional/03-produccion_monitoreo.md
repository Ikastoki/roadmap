# Despliegue, monitoreo y logging en produccion

Desplegar contenedores en produccion va mas alla de ejecutar `docker run` en un servidor. Requiere estrategias de gestion de configuracion, recoleccion de metricas, centralizacion de logs y coordinacion entre multiples hosts. Docker es el punto de partida, pero el ecosistema que lo rodea es lo que convierte un despliegue en un sistema robusto y observable.

## Docker Engine en servidores

Un servidor de produccion que ejecuta Docker debe configurarse con criterios de estabilidad y seguridad. La instalacion directa del Docker Engine sobre una distribucion Linux (Ubuntu Server, Debian, Rocky Linux) sigue siendo la base de muchos despliegues, tanto on-premise como en VM de proveedores cloud.

```bash
# Instalacion del Docker Engine en Ubuntu
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker ubuntu  # solo para usuarios administrativos

# Configuracion del daemon
cat > /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true,
  "userland-proxy": false,
  "iptables": true
}
EOF

systemctl restart docker
```

La opcion `live-restore` permite que los contenedores sigan ejecutandose si el daemon de Docker se reinicia, crucial para evitar ventanas de indisponibilidad durante actualizaciones. Deshabilitar `userland-proxy` reduce el salto de red innecesario cuando se usan las reglas de iptables directamente.

> En produccion jamas se debe exponer el socket de Docker (`/var/run/docker.sock`) a contenedores que no sean de confianza. La herramienta `dockerd` puede configurarse para escuchar en un socket TCP con TLS, pero la practica recomendada es utilizar una plataforma orquestadora que abstraiga el acceso al daemon.

## Plataformas orquestadas

Cuando una aplicacion crece mas alla de un par de servidores, la gestion manual de contenedores se vuelve insostenible. Las plataformas de orquestacion automatizan el despliegue, la escalabilidad, la red y la recuperacion ante fallos.

### Docker Swarm

Swarm es el orquestador nativo de Docker. Convierte un conjunto de nodos Docker en un unico cluster virtual. Su integracion directa con el motor Docker y su curva de aprendizaje reducida lo convierten en una opcion solida para despliegues de tamano medio.

```bash
# Inicializar un swarm
docker swarm init --advertise-addr 10.0.0.1

# Anadir nodos workers
docker swarm join-token worker
docker swarm join --token <token> 10.0.0.1:2377

# Desplegar un stack
docker stack deploy -c compose.prod.yaml myapp
```

Swarm ofrece balanceo de carga integrado (routing mesh), actualizaciones rolling, secretos cifrados y redes superpuestas. Sin embargo, su adopcion ha descendido frente a Kubernetes, principalmente por el menor ecosistema de herramientas y la menor flexibilidad en politicas de escalado.

### Kubernetes

Kubernetes (K8s) es el estandar de facto para orquestacion de contenedores. A diferencia de Swarm, Kubernetes no forma parte de Docker: es una plataforma independiente que puede utilizar cualquier runtime de contenedores (containerd, CRI-O, Docker).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.2.3
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
```

Kubernetes abstrae los conceptos de red, almacenamiento y compute mediante recursos declarativos (Pods, Services, Ingress, PersistentVolumeClaims). Su ecosistema incluye Helm para empaquetado, Prometheus para monitoreo, Istio para service mesh y cientos de operadores que extienden su funcionalidad.

### Nomad

HashiCorp Nomad es una alternativa mas ligera que Kubernetes, disenada para gestionar tanto contenedores como aplicaciones no contenerizadas. Su simplicidad operativa lo hace atractivo para equipos que no necesitan toda la complejidad de K8s.

```hcl
job "myapp" {
  datacenters = ["dc1"]
  group "web" {
    count = 3
    task "server" {
      driver = "docker"
      config {
        image = "registry.example.com/myapp:1.2.3"
        port_map {
          http = 3000
        }
      }
      resources {
        cpu    = 500
        memory = 256
      }
    }
  }
}
```

Nomad se integra de forma nativa con Consul para el descubrimiento de servicios y Vault para la gestion de secretos, formando el ecosistema HashiStack.

## Logging centralizado con Fluentd y Loki

Los logs de un contenedor desaparecen cuando el contenedor se destruye. Para mantener trazabilidad, cada contenedor debe enviar sus logs a un sistema centralizado.

Fluentd es un colector de datos que recibe, transforma y envia logs a multiples destinos. Se despliega como un contenedor o como DaemonSet en Kubernetes.

```bash
# Configuracion del driver de log para enviar a Fluentd
cat > /etc/docker/daemon.json <<EOF
{
  "log-driver": "fluentd",
  "log-opts": {
    "fluentd-address": "localhost:24224",
    "tag": "{{.Name}}"
  }
}
EOF
```

```ruby
# fluentd.conf
<source>
  @type forward
  port 24224
</source>

<filter **>
  @type parser
  key_name log
  format json
</filter>

<match **>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  logstash_format true
</match>
```

Loki, del ecosistema Grafana, ofrece una alternativa mas ligera a Elasticsearch para logs. Indexa unicamente las etiquetas (labels) y no el contenido del mensaje, lo que reduce drasticamente el consumo de almacenamiento.

```yaml
# docker-compose para Loki y Promtail
services:
  loki:
    image: grafana/loki:2.9
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/config.yaml

  promtail:
    image: grafana/promtail:2.9
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers
    command: -config.file=/etc/promtail/config.yaml
```

> La eleccion entre Fluentd + Elasticsearch y Loki depende del volumen de logs y de las necesidades de busqueda. Loki es mas eficiente en recursos y se integra de forma nativa con Grafana, mientras que Elasticsearch ofrece busquedas de texto completo mas potentes.

## Monitoreo con Prometheus y cAdvisor

Prometheus recolecta metricas mediante un modelo pull, raspando endpoints HTTP expuestos por los servicios. Para metricas de infraestructura a nivel de contenedor, cAdvisor (Container Advisor) expone CPU, memoria, red y disco por contenedor.

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "myapp"
    static_configs:
      - targets: ["myapp:3000"]
```

```bash
# Desplegar cAdvisor
docker run -d \
  --name=cadvisor \
  --restart=always \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  gcr.io/cadvisor/cadvisor
```

Prometheus almacena las metricas en una base de datos de series temporales. Grafana se conecta a Prometheus para visualizar dashboards:

```yaml
# docker-compose para el stack de monitoreo
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus-data:
  grafana-data:
```

Un dashboard basico de contenedores incluye: uso de CPU y memoria por contenedor, I/O de disco, trafico de red entrante/saliente, y numero de contenedores en ejecucion. Para aplicaciones, se anaden metricas de negocio: latencia de peticiones, tasa de error, throughput y duracion de eventos.

## Alerting basado en metricas

Prometheus se complementa con Alertmanager para definir reglas que notifiquen cuando las metricas superan umbrales:

```yaml
# alerting-rules.yml
groups:
  - name: container-alerts
    rules:
      - alert: HighMemoryUsage
        expr: sum(container_memory_usage_bytes{container_label_com_docker_compose_service=~".+"}) / sum(container_spec_memory_limit_bytes{container_label_com_docker_compose_service=~".+"}) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Uso de memoria superior al 85%"

      - alert: ContainerRestart
        expr: rate(container_last_seen{container_label_com_docker_compose_service=~".+"}[5m]) < 1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Contenedor reiniciandose repetidamente"
```

Las alertas se encaminan a Slack, PagerDuty, correo electronico o cualquier webhook configurado en Alertmanager.

