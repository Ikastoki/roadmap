# Comparacion de orquestadores

La seleccion de un orquestador no es una decision tecnica aislada: determina el ecosistema de herramientas, el modelo operativo y la portabilidad del sistema a largo plazo. Cada orquestador fue disenado con un contexto especifico en mente, y sus compensaciones reflejan las prioridades de sus autores originales. Kubernetes prioriza la riqueza del ecosistema y la estandarizacion comunitaria sobre la simplicidad. Docker Swarm prioriza la integracion nativa con el ecosistema Docker sobre funcionalidades avanzadas. Nomad prioriza la flexibilidad de workload sobre la especializacion en contenedores. ECS prioriza la integracion nativa con AWS sobre la portabilidad multi-nube. Comprender estas compensaciones es esencial para alinear la herramienta con las necesidades reales del proyecto.

## Kubernetes

Kubernetes nacio en Google como la encarnacion de 15 anos de experiencia orquestando contenedores con Borg y Omega. Es hoy el orquestador de facto de la industria, con soporte nativo en todas las nubes principales (EKS, AKS, GKE) y un ecosistema de herramientas complementarias (Helm, ArgoCD, Istio, Prometheus) que lo convierten en la plataforma mas completa disponible. Su modelo declarativo de estado deseado, basado en recursos YAML extensibles, permite expresar practicamente cualquier patron de despliegue.

La contrapartida es una curva de aprendizaje pronunciada. Los conceptos de Pod, Service, Ingress, StatefulSet, DaemonSet, CronJob, RBAC, NetworkPolicy y los operadores personalizados requieren semanas de estudio para ser comprendidos con rigor. La superficie de errores operativa es amplia: un ClusterRoleBinding mal configurado puede causar fallos de acceso silenciosos que son dificiles de diagnosticar.

```yaml
# Kubernetes: ejemplo de Deployment con servicio y HorizontalPodAutoscaler
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: produccion
  labels:
    app: backend-api
    team: plataforma
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
        - name: api
          image: mi-registry/backend-api:v3.2.1
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 8
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api-svc
  namespace: produccion
spec:
  selector:
    app: backend-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-api-hpa
  namespace: produccion
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

> La disponibilidad de un operador Helm o Kustomize para un servicio no es un indicador de madurez del proyecto, sino una seal de que la comunidad ha adoptado la herramienta como estandar. Verificar la calidad del chart, su frecuencia de actualizacion y el historial de issues es parte de la evaluacion.

## Docker Swarm

Docker Swarm ofrece la experiencia de orquestacion mas cercana a la de un solo host Docker. Con un solo comando se inicializa un cluser y con otro se unen nodos. La compatibilidad total con `docker-compose.yml` significa que un fichero de compose funcional puede desplegarse en un cluser Swarm con minimas modificaciones, lo que lo convierte en la puerta de entrada natural a la orquestacion para equipos que ya dominan Docker.

Sin embargo, Swarm ha recibido poco desarrollo activo desde 2019. Funcionalidades avanzadas como autoscaling automatico, operadores, service meshes integrados o politicas de red avanzadas simplemente no existen o son rudimentarias comparadas con Kubernetes. Para aplicaciones que no requieren estas capacidades, Swarm sigue siendo una solucion solida y significativamente mas simple.

```yaml
# Docker Swarm: despliegue con docker stack deploy
# Fichero stack.yml
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: "0.50"
          memory: 128M
    networks:
      - frontend

  api:
    image: mi-registry/api:latest
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.zone == backend
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
    driver: overlay
    driver_opts:
      encrypted: "true"
```

```bash
# Inicializar Swarm
docker swarm init --advertise-addr 192.168.1.10

# Unir un nodo al cluser
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377

# Desplegar el stack
docker stack deploy -c stack.yml mi-aplicacion

# Ver estado de los servicios
docker stack services mi-aplicacion
```

## Nomad

HashiCorp Nomad se diferencia de los demas orquestadores por su polyglotismo: no solo orquesta contenedores Docker, sino tambien binarios Java, QEMU, raw exec y tareas custom. Esto lo convierte en una herramienta excepcional para entornos heterogeneos donde coexisten aplicaciones containerizadas, binarios compilados y trabajos batch procesados en lote. Nomad usa HCL (HashiCorp Configuration Language) en lugar de YAML, lo que le da una sintaxis mas expresiva pero menos familiar para el ecosistema Kubernetes.

Nomad es significativamente mas simple de operar que Kubernetes. Un servidor unico puede orquestar miles de nodos, y la curva de aprendizaje es considerablemente menor. La integracion nativa con Consul (descubrimiento de servicios) y Vault (gestion de secretos) del mismo ecosistema HashiCorp crea una plataforma coherente cuando se usa el tridente completo.

```hcl
# Nomad: definicion de un job
job "api-backend" {
  datacenters = ["dc1"]
  type        = "service"

  group "produccion" {
    count = 3

    network {
      port "http" { to = 8080 }
    }

    task "servidor" {
      driver = "docker"

      config {
        image = "mi-registry/backend-api:v3.2.1"
        ports = ["http"]
      }

      resources {
        cpu    = 500
        memory = 256
      }

      service {
        name     = "api-backend"
        port     = "http"
        tags     = ["produccion", "v3"]

        check {
          type     = "http"
          path     = "/health"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

```bash
# Desplegar el job
nomad job run api-backend.nomad.hcl

# Verificar el estado
nomad job status api-backend

# Escalar manualmente
nomad job scale api-backend produccion 5
```

## Amazon ECS

Elastic Container Service esta profundamente integrado con el ecosistema AWS: ALB, CloudWatch, IAM, EC2 o Fargate para ejecucion serverless. La eliminacion de la gestion del cluser (cuando se usa Fargate) reduce la carga operativa a cero en cuanto a infraestructura subyacente, lo que lo convierte en la opcion mas rapida para equipos全 dedicados a AWS.

La principal limitacion es el acoplamiento al proveedor. Las definiciones de tarea ECS no son portables a otros entornos, las politicas de red dependen de VPC y subnets de AWS, y funcionalidades como service discovery usan Cloud Map. Para equipos multi-cloud o con requisitos de portabilidad, esta dependencia es un costo significativo.

```json
// ECS Task Definition (simplificada)
{
  "family": "backend-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "mi-registry/backend-api:v3.2.1",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 15,
        "timeout": 5,
        "retries": 3
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/backend-api",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## Apache Mesos (y Marathon)

Mesos fue el orquestador que impulso Twitter, Apple (Siri) y Uber en sus primeros anos. Su arquitectura de dos niveles (Mesos como kernel distribuido, Marathon como framework de orquestacion de contenedores) le permitia gestionar clusters de cientos de miles de nodos con una granularidad de recursos fina. Sin embargo, desde que Mesosphere fue adquirida por D2iQ y el ecosistema se fragmento con DC/OS, la comunidad se ha reducido significativamente. Hoy se considera una solucion legacy, aunque partes de su arquitectura influyeron directamente en el diseño de Kubernetes.

## Tabla comparativa detallada

| Caracteristica | Kubernetes | Docker Swarm | Nomad | ECS (Fargate) | Mesos/Marathon |
|---|---|---|---|---|---|
| **Curva de aprendizaje** | Alta | Baja | Media | Baja-media | Alta |
| **Workloads soportados** | Contenedores | Contenedores | Contenedores, binarios, VMs, batch | Contenedores (Docker/OCI) | Contenedores via framework |
| **Escalado automatico** | HPA, VPA, KEDA, Cluster Autoscaler | No nativo (externo con scripts) | Autoscaler externo | Application Auto Scaling | Framework-specific |
| **Multi-cloud/portabilidad** | Alta (estandar CNCF) | Media | Alta | Baja (acoplado a AWS) | Media |
| **Ecosistema** | Muy amplio (Helm, ArgoCD, Istio, Prometheus) | Limitado | Medio (Consul, Vault) | Integrado AWS | Pequeno |
| **Gestion de secretos** | Secrets nativos + Vault | Docker Secrets | Vault (integracion nativa) | Secrets Manager + IAM | Vault |
| **Rolling update** | Nativo, configurable | Nativo, basico | Nativo, configurable | Nativo (ECS service) | Nativo via Marathon |
| **Service mesh** | Istio, Linkerd, Cilium | No disponible | Consul Connect | App Mesh (descontinuado) | No disponible |
| **Produccion activa** | Muy activo (CNCF) | Mantenimiento minimo | Activo (HashiCorp) | Muy activo (AWS) | Legacy |
| **Costo infraestructura base** | Alto (3+ nodos para HA) | Bajo (2+ nodos) | Medio (3+ servidores) | Cero (Fargate) | Alto (ZooKeeper + Mesos + Marathon) |

> La tendencia del mercado es clara: Kubernetes domina el 96%+ de los entornos de orquestacion en produccion segun la encuesta CNCF 2024. Elegir una alternativa a Kubernetes es legitimo, pero implica asumir que el ecosistema de herramientas sera mas reducido y que la contratacion de talento especializado sera mas dificil.

