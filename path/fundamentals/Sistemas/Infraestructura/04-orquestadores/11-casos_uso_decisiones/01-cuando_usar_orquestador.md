# Cuando usar un orquestador

Un orquestador de contenedores introduce una capa de abstraccion significativa sobre el runtime de contenedores basicos. No se trata solo de ejecutar contenedores en un cluser, sino de delegar al sistema decisiones como programacion, balanceo de carga, resiliencia y descubrimiento de servicios. Sin embargo, esta potencia conlleva una complejidad operativa que no siempre esta justificada. La decision de adoptar un orquestador debe basarse en una evaluacion honesta de la complejidad real de la aplicacion, los requisitos de disponibilidad, el tamano del equipo y la velocidad a la que cambian los requerimientos. Un error comun es introducir Kubernetes en un entorno donde Docker Compose o una plataforma gestionada como Heroku resolverian el problema con menos de una decima parte del esfuerzo operativo.

## Escenarios donde un orquestador es la opcion correcta

La adopcion de un orquestador esta justificada cuando el sistema presenta varias de estas caracteristicas simultaneamente:

- **Microservicios con dependencias complejas.** Cuando una aplicacion se compone de multiples servicios independientes que deben comunicarse, escalarse y desplegarse de forma separada, un orquestador gestiona automaticamente el descubrimiento entre servicios, el balanceo de carga interno y las politicas de reinicio.
- **Escalado dinamico e irregular.** Si el trafico presenta picos impredecibles o estacionalidad marcada, el escalado horizontal automatico del orquestador permite absorber carga sin intervencion manual ni provisionamiento permanente de infraestructura.
- **Multi-equipo con despliegues concurrentes.** En organizaciones donde varios equipos despliegan servicios distintos en la misma infraestructura compartida, el orquestador ofrece namespacing, politicas de recursos y aislamiento que evitan conflictos.
- **Disponibilidad alta con tolerancia a fallos.** Cuando la aplicacion requiere uptime del 99.9% o superior, el orquestador redistribuye cargas automaticamente ante fallos de nodos, reprograma contenedores y mantiene replicas suficientes segun la declaracion del estado deseado.

```yaml
# Ejemplo: politica de tolerancia a fallos en Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-gateway
      containers:
        - name: gateway
          image: mi-registry/api-gateway:2.1.0
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 3
```

## Escenarios donde un orquestador es innecesario

- **Aplicaciones monoliticas sencillas.** Un monolito desplegado como un unico contenedor o par de contenedores (app + base de datos) no necesita la complejidad de un cluser orquestado. Docker Compose en un solo host o una VM con systemd son soluciones proporcionales.
- **Equipos pequenos sin experiencia en operaciones distribuidas.** Un equipo de dos o tres desarrolladores puede dedicar mas tiempo manteniendo el cluser de Kubernetes que desarrollando funcionalidad de negocio. El costo de aprendizaje y operacion continua es real.
- **Cargas de trabajo estaticas y predecibles.** Si el trafico es constante, el numero de usuarios no crece significativamente y no hay picos, el provisionamiento manual de un par de servidores es suficiente y mas economico.

> Un servidor bien dimensionado ejecutando Docker Compose con reinicio automatico (`restart: always`) ofrece resistencia a fallos basica sin la sobrecarga operativa de un cluser distribuido.

## Alternativas practicas

| Necesidad | Alternativa al orquestador | Complejidad operativa |
|---|---|---|
| Desarrollo local multi-servicio | Docker Compose | Baja |
| Despliegue rapido sin infraestructura | PaaS (Heroku, Railway, Render) | Muy baja |
| Balanceo de carga basico | Nginx/HAProxy + Docker | Baja |
| Escalado simple en la nube | Auto Scaling Groups (AWS) / VM Scale Sets (Azure) | Media-baja |
| Despliegue en un solo servidor | Docker Swarm (modo single-node) | Baja |

```yaml
# Docker Compose: solucion suficiente para aplicaciones simples
version: "3.9"
services:
  web:
    build: .
    ports:
      - "80:8000"
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: always
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
volumes:
  pgdata:
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

## Arbol de decision

El siguiente diagrama resume el proceso de decision. Cada nodo representa una pregunta binaria cuya respuesta dirige hacia la solucion mas adecuada para el nivel de complejidad del proyecto:

```
                        ┌─────────────────────────┐
                        │  Tienes multiples        │
                        │  servicios a desplegar?  │
                        └───────────┬─────────────┘
                           No ──────┤────── Si
                            │                │
                            ▼                ▼
                   ┌────────────┐   ┌─────────────────────────┐
                   │  ¿Crecimiento│   │  ¿Requiere escalado     │
                   │  esperado?  │   │  dinamico o HA?         │
                   └─────┬──────┘   └───────────┬─────────────┘
                  No ────┤─── Si        No ──────┤────── Si
                   │     │           │                │
                   ▼     ▼           ▼                ▼
              ┌────────┐┌─────────┐ ┌──────────┐  ┌──────────────┐
              │Monolito││  PaaS   │ │Docker    │  │ ORQUESTADOR  │
              │directo ││(Heroku) │ │Compose + │  │ (K8s, Swarm, │
              │systemd ││         │ │autoscaler│  │  Nomad)      │
              └────────┘└─────────┘ └──────────┘  └──────────────┘

        ┌──────────────────────────────────────┐
        │  Si equipo >= 5 devs Y microservicios │
        │  Y despliegues frecuentes             │
        │  → Kubernetes                         │
        │                                       │
        │  Si equipo < 5 devs Y simplicidad     │
        │  prioritaria                          │
        │  → Docker Swarm o PaaS gestionado     │
        └──────────────────────────────────────┘
```

> No existe una respuesta universal. Un equipo que ya domina Kubernetes puede razonablemente usarlo incluso para aplicaciones simples, aprovechando la infraestructura existente. La clave es evaluar si la complejidad del orquestador genera valor real en el contexto del proyecto, no si es inherentemente superior.

