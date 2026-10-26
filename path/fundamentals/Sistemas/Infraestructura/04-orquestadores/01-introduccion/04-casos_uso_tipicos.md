# Casos de uso tipicos

La orquestacion de contenedores no es una solucion universal para todos los problemas de despliegue; es una herramienta que encaja especialmente bien en escenarios donde la escalabilidad, la resiliencia y la gestion centralizada son requisitos criticos. Comprender estos casos de uso permite evaluar cuándo una plataforma de orquestacion aporta valor real y cuando una solucion mas simple es suficiente. Los escenarios siguientes representan los patrones mas comunes en la industria, cada uno con necesidades especificas que justifican la adopcion de un orquestador.

## Microservicios distribuidos

Una arquitectura de microservicios descompone una aplicacion monolitica en multiples servicios independientes que se comunican entre si a traves de APIs. Cada servicio tiene su propio ciclo de vida, su propia base de datos y su propia velocidad de despliegue. Gestionar docenas o centenas de estos servicios de forma manual es inviable: cada uno requiere escalado independiente, actualizaciones continuas y tolerancia a fallos individual. La orquestacion permite definir cada microservicio como un Deployment con su Service asociado, gestionar la comunicacion entre ellos a traves de mesh de servicios y desplegar actualizaciones de un servicio sin afectar a los demas.

```yaml
# Despliegue de un sistema de microservicios completo
# Servicio de autenticacion
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
      - name: auth
        image: myapp/auth-service:1.3.0
        ports:
        - containerPort: 8081
        env:
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-secrets
              key: jwt-key
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

---
# Servicio de procesamiento de pedidos
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
      - name: orders
        image: myapp/orders-service:2.0.1
        ports:
        - containerPort: 8082
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: infra-config
              key: postgres-host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
        readinessProbe:
          httpGet:
            path: /health
            port: 8082
          initialDelaySeconds: 10
          periodSeconds: 5

---
# Service para cada microservicio
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth
  ports:
  - port: 80
    targetPort: 8081

---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
  - port: 80
    targetPort: 8082
```

> Cuando el numero de microservicios supera las 20-30 unidades, el valor de la orquestacion se multiplica. Para menos de 5 servicios en un unico servidor, Docker Compose sigue siendo una alternativa valida y mucho mas simple.

## Plataformas SaaS multi-tenant

Las plataformas SaaS sirven a multiples clientes (tenants) simultaneamente sobre la misma infraestructura. Cada tenant puede requerir aislamiento de datos, configuracion personalizada y niveles de servicio diferentes. La orquestacion facilita el aislamiento mediante namespaces separados, la asignacion de recursos diferenciada por tenant y el escalado independiente segun la demanda de cada cliente. Los namespaces permiten que un tenant problematico no consuma todos los recursos del clúster, mientras que los Network Policies garantizan el aislamiento de red entre tenants.

```yaml
# Namespace aislado para un tenant premium
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-empresa-abc
  labels:
    tenant: empresa-abc
    tier: premium

---
# ResourceQuota diferenciado por tier
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-empresa-abc
  namespace: tenant-empresa-abc
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    pods: "15"

---
# NetworkPolicy que aislamiento entre tenants
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-tenant
  namespace: tenant-empresa-abc
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingresses:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: empresa-abc
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: empresa-abc

---
# Despliegue del servicio del tenant con limites diferenciados
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-principal
  namespace: tenant-empresa-abc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: plataforma-saas
  template:
    metadata:
      labels:
        app: plataforma-saas
    spec:
      containers:
      - name: app
        image: miapp/plataforma:4.2.0
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

## Jobs por lotes y tareas programadas

No todas las cargas de trabajo son servicios de larga duracion. Muchas organizaciones ejecutan procesos batch que se ejecutan periodicamente: generacion de reportes, procesamiento de imagenes, migraciones de datos, entrenamiento de modelos de machine learning y sincronizaciones de sistemas. Estos jobs tienen requisitos diferentes a los servicios web: necesitan ejecutarse una vez (o periodicamente), consumir grandes cantidades de recursos temporalmente y terminar automaticamente al completarse. Los orquestadores proporcionan tipos de trabajo especificos para estos escenarios.

```yaml
# CronJob en Kubernetes que ejecuta un reporte diario
apiVersion: batch/v1
kind: CronJob
metadata:
  name: generador-reportes
spec:
  schedule: "0 2 * * *"  # Todos los dias a las 2:00 AM
  jobTemplate:
    spec:
      completions: 1
      backoffLimit: 3
      activeDeadlineSeconds: 3600
      template:
        spec:
          containers:
          - name: reporte
            image: miapp/report-generator:1.0.0
            env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: REPORT_TYPE
              value: "daily-sales"
            resources:
              requests:
                cpu: "1"
                memory: "2Gi"
              limits:
                cpu: "2"
                memory: "4Gi"
          restartPolicy: OnFailure

---
# Job de procesamiento batch paralelo
apiVersion: batch/v1
kind: Job
metadata:
  name: procesar-cola
spec:
  parallelism: 5    # 5 workers en paralelo
  completions: 100  # 100 tareas totales
  template:
    spec:
      containers:
      - name: worker
        image: miapp/batch-processor:1.0.0
        env:
        - name: QUEUE_URL
          value: "redis://redis-service:6379"
        - name: BATCH_SIZE
          value: "50"
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
      restartPolicy: Never
  backoffLimit: 2
```

## Entornos de CI/CD con multiples etapas

Los pipelines de integracion y despliegue continuo requieren entornos aislados para cada etapa del ciclo de desarrollo: desarrollo, pruebas, staging y produccion. La orquestacion permite levantar y destruir entornos completos de forma efimera, ejecutar tests en paralelo sobre infraestructura identica a produccion, y desplegar de forma segura promoviendo imagenes entre entornos. Cada namespace representa una etapa del pipeline, con configuraciones y restricciones propias.

```yaml
# Entorno de staging: replica reducida de produccion
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-staging
  namespace: staging
  labels:
    environment: staging
    pipeline: ci-cd
spec:
  replicas: 1  # Una sola instancia para ahorrar recursos
  selector:
    matchLabels:
      app: api
      environment: staging
  template:
    metadata:
      labels:
        app: api
        environment: staging
    spec:
      containers:
      - name: api
        image: myapp/api:${IMAGE_TAG}  # Inyectado por el pipeline
        ports:
        - containerPort: 8080
        env:
        - name: ENV
          value: "staging"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: staging-config
              key: db-host
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3

---
# Entorno de produccion: alta disponibilidad
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-production
  namespace: production
  labels:
    environment: production
    pipeline: ci-cd
spec:
  replicas: 5  # Multi-instancia con anti-afinidad
  selector:
    matchLabels:
      app: api
      environment: production
  template:
    metadata:
      labels:
        app: api
        environment: production
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: myapp/api:${IMAGE_TAG}
        ports:
        - containerPort: 8080
        env:
        - name: ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: prod-config
              key: db-host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: prod-secrets
              key: db-password
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

---
# HPA: escalado automatico basado en uso de CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-production
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

> Los entornos efimeros de CI/CD son uno de los casos de uso que mas beneficio reportan. Levantar un entorno completo de staging en 30 segundos y destruirlo al terminar el pipeline reduce costes de infraestructura y elimina problemas de contaminacion entre pruebas.

