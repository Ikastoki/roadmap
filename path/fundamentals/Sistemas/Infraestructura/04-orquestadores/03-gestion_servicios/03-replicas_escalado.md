# Replicas y escalado

La capacidad de ejecutar multiples instancias simultaneas de un servicio es uno de los pilares fundamentales de la orquestacion de contenedores. Las replicas representan el numero de copias identicas de un Pod que deben estar ejecutandose en cualquier momento dado. El orquestador no solo mantiene ese numero, sino que lo vigila continuamente: si un Pod muere por un fallo de hardware, un OOMKill o un error en la aplicacion, el controlador detecta la discrepancia entre el estado deseado (el numero de replicas definido) y el estado real (el numero de Pods eficazmente en ejecucion) y programa automaticamente un nuevo Pod para restaurar la consistencia. Este ciclo de reconciliacion continua es lo que otorga resiliencia a los servicios desplegados en un clúster.

El escalado es el mecanismo por el cual se ajusta el numero de replicas para responder a cambios en la demanda. Existen dos tipos principales: el escalado manual y el escalado automatico. El escalado manual implica una decision humana o programada que modifica explicitamente el campo `replicas` del recurso. Es util en escenarios donde el patron de carga es predecible: por ejemplo, aumentar replicas antes de un evento de alto trafico programado o reducirlas durante ventanas de mantenimiento. El escalado automatico, representado por el Horizontal Pod Autoscaler (HPA) en Kubernetes, monitoriza metricas en tiempo real (consumo de CPU, memoria, metricas personalizadas como longitud de cola de un mensaje) y ajusta las replicas dinamicamente segun una logica de control proporcional.

### Escalado manual con kubectl

```bash
# Escalar un Deployment a 5 replicas
kubectl scale deployment api-gateway --replicas=5 -n produccion

# Verificar que las nuevas replicas estan en estado Running
kubectl get pods -l app=api-gateway -n produccion -w

# Reducir a 2 replicas (por ejemplo, fuera de horario laboral)
kubectl scale deployment api-gateway --replicas=2 -n produccion

# Escalar basado en una definicion declarativa
kubectl apply -f deployment.yaml
```

### Horizontal Pod Autoscaler (HPA)

El HPA funciona mediante un ciclo de control que se ejecuta cada 15 segundos por defecto. En cada ciclo, el controlador recoge las metricas del Metrics Server (o un adaptador de metricas personalizado), calcula la relacion entre el valor actual y el valor objetivo, y ajusta el numero de replicas proporcionalmente. La formula basica es: `replicas_deseadas = ceil(replicas_actuales * (valor_actual_metrica / valor_objetivo_metrica))`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
  namespace: produccion
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 120
```

> El `stabilizationWindowSeconds` en el bloque `behavior` es critico para evitar el fenomeno de "oscilacion" (thrashing), donde el HPA escala sube y baja repetidamente porque las metricas fluctuan rapidamente. Una ventana de estabilizacion de 300 segundos en el escalado hacia abajo asegura que la carga haya disminuido de forma sostenida antes de reducir replicas.

El bloque `behavior` permite definir politicas asimetricas: escalar rapido hacia arriba (para absorber picos de carga) pero lento hacia abajo (para evitar que un servicio se quede sin capacidad si la carga vuelve a subir). Los tipos de politica disponibles son `Percent` (porcentaje del valor actual) y `Pods` (un numero fijo de replicas) y `Containers` (basado en un recurso especifico de contenedor).

> El HPA requiere que los contenedores declaren `resources.requests` en su definicion. Sin estos valores, el Metrics Server no puede calcular el porcentaje de utilizacion y el HPA no funcionara. Si no se especifican requests en el YAML, el HPA quedara en estado `UNABLE_TO_SCALE` sin errores visibles inmediatos, lo que puede pasar desapercibido hasta que se produzca un pico de carga no gestionado.

```bash
# Ver el estado del HPA
kubectl get hpa -n produccion
# NAME              REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS
# api-gateway-hpa   Deployment/api-gateway       45%/70%, 60%/80%   2         20        3

# Describir el HPA para ver historial de escalados
kubectl describe hpa api-gateway-hpa -n produccion
```

### Escalado con metricas personalizadas

Para servicios cuya carga no se correlaciona con CPU o memoria (como una cola de mensajes donde la metrica relevante es el numero de mensajes pendientes), se puede utilizar un adaptador de metricas personalizado que exponga metricas a traves de la API de metricas de Kubernetes:

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: cola_mensajes_pendientes
      target:
        type: AverageValue
        averageValue: "50"
```

