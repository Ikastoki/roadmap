# Migracion a orquestadores

Migrar una aplicacion existente a un orquestador de contenedores no es un evento puntual sino un proceso iterativo que requiere evaluacion, planificacion y ejecucion gradual. El error mas frecuente es intentar migrar todo simultaneamente: convertir la aplicacion a contenedores, configurar el cluser y cambiar la infraestructura de produccion en un unico sprint. Esta aproximacion multiplica los puntos de fallo y dificulta la identificacion de problemas. Un enfoque por fases, donde cada paso se valida antes de avanzar al siguiente, reduce el riesgo y permite al equipo adquirir experiencia incrementalmente.

## Evaluacion inicial del estado

Antes de iniciar la migracion, el equipo debe responder preguntas concretas sobre la aplicacion actual, la infraestructura existente y los recursos disponibles. Esta evaluacion determina no solo si la migracion es viable, sino tambien que estrategia segura:

- **Complejidad de la aplicacion.** Un monolito con una base de datos relacional y un frontend estatico tiene un camino de migracion diferente al de una arquitectura de microservicios con comunicacion asincrona via colas. El monolito podria beneficiarse de un refactoring previo antes de contenedorizarse.
- **Dependencias de estado.** Las aplicaciones con estado persistente (bases de datos, colas de mensajes con archivos, caches distribuidos) requieren estrategias especificas de almacenamiento persistente en el orquestador. No todas las soluciones de almacenamiento son iguales en cada plataforma.
- **Equipo disponible.** Migrar a Kubernetes requiere al menos una persona con conocimientos solidos de la herramienta. Si el equipo no tiene experiencia, la inversion en formacion debe formar parte del plan con un presupuesto de tiempo realista.

```bash
# Script de evaluacion: analisis de dependencias de una aplicacion Node.js
# Identificar puertos expuestos, variables de entorno y dependencias externas

echo "=== Puertos expuestos ==="
grep -r "listen\|PORT" src/ --include="*.js" -h | sort -u

echo "=== Variables de entorno ==="
grep -r "process.env\." src/ --include="*.js" -ho | sort -u

echo "=== Conexiones a bases de datos ==="
grep -rn "mongoose\|sequelize\|pg\|mysql\|redis" src/ --include="*.js" | head -20

echo "=== Servicios externos consumidos ==="
grep -rn "axios\|fetch\|http://" src/ --include="*.js" | grep -v "localhost" | head -20

echo "=== Sistema de archivos ==="
grep -rn "fs\.\|writeFile\|readFile\|createReadStream" src/ --include="*.js" | head -10

echo "=== Archivos de configuracion ==="
find . -name ".env*" -o -name "config.*" -o -name "*.yml" -o -name "*.yaml" | head -10
```

## Estrategia de migracion gradual

La estrategia mas segura es la migracion incremental, donde los servicios se migran uno a uno mientras la infraestructura antigua y la nueva coexisten durante un periodo transitorio. Esto permite validar cada componente en produccion sin comprometer la disponibilidad del sistema completo.

### Fase 1: Contenedorizacion de servicios

El primer paso es convertir cada servicio a contenedores Docker, idealmente sin cambios en la arquitectura. Esta fase valida que la aplicacion funciona correctamente dentro de un contenedor y expone problemas de configuracion (paths hardcodeados, dependencias del sistema operativo host, archivos de estado en directorios que no son un volumen).

```dockerfile
# Dockerfile optimizado para una aplicacion Java Spring Boot
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN addgroup -S app && adduser -S app -G app

WORKDIR /opt/app

COPY --chown=app:app target/api-*.jar app.jar

USER app

EXPOSE 8080

HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

### Fase 2: Configuracion del cluser de destino

El cluser se configura durante esta fase, pero sin desplegar servicios de produccion aun. Se instalan los componentes basicos (ingress controller, monitorizacion, logging) y se validan las politicas de red, seguridad y almacenamiento. Esta fase es donde el equipo adquiere experiencia operativa con el orquestador en un entorno de bajo riesgo.

```yaml
# Namespace de staging para validar antes de produccion
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
    istio-injection: enabled
---
# Politica de red que permite trafico solo desde el ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-only
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
```

### Fase 3: Despliegue progresivo en produccion

Cada servicio se despliega individualmente en produccion, comenzando por los menos criticos. La validacion en produccion real es indispensable: problemas de latencia, patrones de trafico y comportamiento bajo carga no se replican fielmente en staging. Cada servicio migra, se valida con trafico real durante un periodo de estabilizacion, y solo entonces se desactiva la instancia anterior.

```yaml
# Estrategia de migracion con canary deployment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-api
  namespace: produccion
spec:
  hosts:
    - api.miempresa.com
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: backend-api-nuevo
            port:
              number: 80
    - route:
        - destination:
            host: backend-api-antiguo
            port:
              number: 80
          weight: 90
        - destination:
            host: backend-api-nuevo
            port:
              number: 80
          weight: 10
```

### Fase 4: Consolidacion y limpieza

Una vez que todos los servicios funcionan en el orquestador y la infraestructura anterior no recibe trafico, se procede a la desactivacion y eliminacion de los recursos legacy. Esta fase incluye la eliminacion de servidores antiguos, la migracion de datos persistentes si es necesario y la actualizacion de la documentacion de la infraestructura.

## Adaptacion a patrones cloud-native

Los principios de la metodologia 12-Factor App son directamente aplicables a la migracion a orquestadores. Algunos factores son criticos para el correcto funcionamiento en un entorno orquestado:

- **Factor 3: Configuracion en variables de entorno.** Los secrets y configuracion especifica del entorno no deben codificarse en la imagen. En Kubernetes, esto se implementa con Secrets y ConfigMaps montados como variables de entorno o ficheros.
- **Factor 6: Proceso sin estado.** Los contenedores deben ser efimeros y reemplazables. Cualquier estado persistente debe almacenarse en servicios externos (bases de datos, S3, Redis).
- **Factor 11: Logs como streams de eventos.** Los logs se escriben a stdout/stderr y el runtime del orquestador se encarga de su recogida y enrutamiento.
- **Factor 12: Procesos como servicios descubribles.** Cada servicio debe exponer un endpoint de salud y ser identificable por nombre en la red interna del cluser.

```yaml
# Ejemplo de Secret y ConfigMap para un servicio migrado
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: produccion
data:
  DATABASE_HOST: "db-proxy.produccion.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "app_production"
  LOG_LEVEL: "info"
  CACHE_TTL: "300"
---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: produccion
type: Opaque
data:
  DATABASE_USERNAME: YXBwX3VzZXI=    # base64 codificado
  DATABASE_PASSWORD: c3VwZXJzZWNyZXQ= # base64 codificado
  JWT_SECRET: bWlzZWNyZXRvYXV0aA==    # base64 codificado
```

## Lista de verificacion de migracion

Antes de desplegar cada servicio en produccion, el equipo debe validar los siguientes puntos:

- [ ] La imagen Docker se construye de forma reproducible y no depende de caches externos
- [ ] La imagen usa un usuario no-root y un base image actualizado y escaneado
- [ ] La configuracion se inyecta via variables de entorno, no via archivos baked en la imagen
- [ ] Los secrets se almacenan en el gestor de secretos del orquestador, no en el repositorio
- [ ] Los endpoints de salud (liveness y readiness) estan implementados y responden correctamente
- [ ] Los logs se escriben a stdout/stderr y no a ficheros locales
- [ ] Los recursos CPU y memoria estan declarados con requests y limits realistas
- [ ] La estrategia de despliegue (rolling update, canary) esta configurada y probada
- [ ] El monitoreo (metricas, logs, trazas) funciona correctamente para el servicio
- [ ] Los backups de datos persistentes estan configurados y verificados

> La migracion mas exitosa es aquella donde cada paso es un commit que puede revertirse. Si un servicio no funciona correctamente en el orquestador, el trafico se redirige a la instancia anterior sin impacto al usuario final. Esta reversibilidad es la garantia de seguridad del proceso.

