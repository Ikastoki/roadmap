# Limites y el siguiente paso

Docker y los motores de contenedores locales son excelentes para desarrollo, pruebas y despliegues simples, pero tienen limitaciones claras cuando la aplicacion crece. No ofrecen autoescalado, auto-reparacion, despliegues complejos ni alta disponibilidad. Para esas necesidades, la transicion natural es hacia orquestadores como Kubernetes.

## Limitaciones del motor local

```bash
# Docker no autoescalado
docker run -d --name app mi-app
# Si el contenedor falla, no se reinicia automaticamente (a menos que --restart)
# Si hay trafico alto, no crea mas instancias
# Si el host falla, los contenedores se pierden

# Docker no orquesta multi-host
# No hay comunicacion nativa entre hosts
# No hay balanceo de carga automatico
# No hay despliegues con cero downtime nativos
```

## Cuando escalar a Kubernetes

- La aplicacion tiene multiples servicios que deben escalar independentemente.
- Se necesita alta disponibilidad (replicas en multiples nodos).
- Se requieren despliegues con estrategias (rolling update, blue-green, canary).
- Hay necesidad de autoescalado basado en metricas.
- La complejidad operativa justifica la inversicion en orquestacion.

## Transicion gradual

```bash
# Paso 1: Docker Compose en un servidor
docker compose up -d

# Paso 2: Docker Swarm (cluster simple)
docker swarm init
docker stack deploy -c compose.yml mi-app

# Paso 3: Kubernetes (cluster completo)
kubectl apply -f k8s/
```

## Docker Swarm (paso intermedio)

```bash
# Inicializar swarm
docker swarm init --advertise-addr 192.168.1.100

# Desplegar stack
docker stack deploy -c compose.yml mi-app

# Ver servicios
docker stack services mi-app

# Escalar
docker service scale mi-app_web=3
```

## Kubernetes (produccion)

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: mi-app
          image: miusuario/mi-app:v1.2.3
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: mi-app
spec:
  selector:
    app: mi-app
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer
```

## Resumen del recorrido

```
Docker basico (contenedores individuales)
    │
    ▼
Docker Compose (multi-contenedor local)
    │
    ▼
Docker Swarm (cluster simple)
    │
    ▼
Kubernetes (orquestacion completa)
```

> Los contenedores son el fundamento de la infraestructura moderna, pero no son la solucion final. A medida que la complejidad de la aplicacion crece, la necesidad de orquestacion, autoescalado y alta disponibilidad se vuelve inevitable. Kubernetes es el orquestador de referencia en la industria, y el conocimiento de Docker es el prerrequisito indispensable para dominarlo.

