# Definicion declarativa de servicios

Un orquestador moderno no funciona como un orquestador de tareas clásico que ejecuta comandos de forma imperativa. En su lugar, adopta un modelo declarativo: el operador describe el estado deseado del sistema y el orquestador se encarga de calcular y ejecutar los pasos necesarios para alcanzarlo. Esta distinción es fundamental. En un enfoque imperativo, se le indica al sistema "crea tres contenedores con esta imagen"; en uno declarativo, se define "debe existir un servicio compuesto por tres réplicas de esta imagen con estos recursos", y es el motor de conciliación del orquestador el que verifica continuamente si el estado real coincide con el declarado, tomando acciones correctivas cuando no es así. Este patrón de "reconciliación continua" es el corazón de sistemas como Kubernetes, Docker Swarm o Nomad, y es lo que permite la resiliencia, la auto-recuperación y la consistencia en entornos distribuidos a gran escala.

La definición declarativa de un servicio se estructura en un documento que sigue un esquma conocido. Los componentes típicos son: la versión de la API que se utiliza para interpretar el documento, el tipo de recurso que se está declarando (Deployment, StatefulSet, DaemonSet, Job), un bloque de metadatos con nombre, etiquetas y anotaciones, y un bloque de especificación (spec) que describe cómo debe ser el recurso: qué imagen ejecutar, cuántas réplicas mantener, qué recursos consumir, qué estrategia de actualización aplicar, entre otros detalles. Los formatos más habituales para estos documentos son YAML (el más extendido por su legibilidad), JSON (común en APIs REST) y HCL (usado por Terraform y Vault). Cada uno tiene sus ventajas, pero YAML domina en el ecosistema de orquestadores de contenedores por su balance entre claridad y concisión.

Un despliegue declarativo completo en Kubernetes incluye no solo la configuración funcional del servicio, sino también metadatos enriquecidos que permiten organizacion, monitorización y control de acceso. Las anotaciones, por ejemplo, son pares clave-valor que almacenan información complementaria no selectora: nombres de equipo responsable, enlaces a documentación interna, configuración de herramientas de CI/CD o indicadores para controladores de ingress.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: produccion
  labels:
    app: api-gateway
    version: v2.3.1
    tier: frontend
  annotations:
    owner: "equipo-platform@empresa.com"
    docs: "https://wiki.empresa.com/api-gateway"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api-gateway
        version: v2.3.1
    spec:
      containers:
        - name: gateway
          image: registry.empresa.com/api-gateway:v2.3.1
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          env:
            - name: LOG_LEVEL
              value: "info"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: api-config
                  key: database-host
```

> Las definiciones declarativas son idempotentes: aplicar el mismo documento múltiples veces produce el mismo resultado. Esto permite integrarlas en pipelines de CI/CD de forma segura, ya que `kubectl apply` puede ejecutarse repetidamente sin efectos colaterales no deseados.

La selección de etiquetas (`selector`) en el bloque spec es lo que vincula un Deployment con los Pods que gestiona. Si las etiquetas del template no coinciden con el selector, el controlador rechazará la definición. Este mecanismo de selección basado en etiquetas es el mismo que utilizan los Servicios (Service) para enrutar tráfico hacia un conjunto de Pods, lo que convierte a las etiquetas en el sistema de referenciación más importante de Kubernetes.

