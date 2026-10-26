# Diseno de aplicaciones para orquestadores

Los orquestadores de contenedores no son simples ejecutores de procesos: asumen control total sobre el ciclo de vida de las aplicaciones, incluyendo su creacion, destruccion, redes y almacenamiento. Esta asuncion de responsabilidad obliga a repensar el diseno de las aplicaciones desde sus cimientos. Una aplicacion concebida para ejecutarse en un servidor fisico o una VM tradicional traslada implicitamente supuestos sobre persistencia de estado, configuracion local y disponibilidad de servicios auxiliares que colisionan directamente con el modelo de orquestacion. El orquestador espera que las aplicaciones sean desechables, reemplazables y reproducibles; si la aplicacion no cumple estos requisitos, el orquestador pierde gran parte de su valor y puede incluso empeorar la estabilidad del sistema.

La metodologia Twelve-Factor App, originalmente formulada para aplicaciones en la nube, se ha convertido en el marco de referencia ineludible para disenar aplicaciones compatibles con orquestadores. Sus principios no son recomendaciones esteticas sino restrucciones arquitectonicas que permiten al orquestador gestionar las instancias de forma eficiente y predecible.

## Metodologia Twelve-Factor

El metodo Twelve-Factor define diez reglas que, en conjunto, garantizan que una aplicacion pueda ejecutarse de forma idonea en un entorno orquestado. No todas estan directamente relacionadas con el orquestador, pero la mayoria son prerrequisitos para obtener el maximo beneficio de la orquestacion.

**I. Codebase**: una unica aplicacion en un repositorio, desplegada multiples veces (produccion, staging, desarrollo).

**II. Dependencies**: declarar todas las dependencias de forma explicita. Nunca depender de paquetes del sistema subyacente.

**III. Config**: almacenar la configuracion en variables de entorno, nunca en el codigo fuente.

**IV. Backing services**: tratar servicios auxiliares (bases de datos, colas, SMTP) como recursos adjuntos sin distincion entre locales y remotos.

**V. Build, release, run**: separar estrictamente las fases de compilacion, empaquetado y ejecucion.

**VI. Processes**: ejecutar la aplicacion como uno o mas procesos sin estado. El estado debe persistirse en servicios externos.

**VII. Port binding**: la aplicacion se expone como un servicio HTTP (u otro protocolo) a traves del binding a un puerto, sin depender de un servidor web externo.

**VIII. Concurrency**: escalar horizontalmente mediante procesos adicionales, no mediante hilos.

**IX. Disposability**: los procesos deben iniciarse rapido y apagarse limpiamente, maximizando la tolerancia a fallos.

**X. Dev/prod parity**: mantener la mayor similitud posible entre entornos de desarrollo y produccion.

**XI. Logs**: tratar los logs como un stream de eventos. Nunca gestionar su rotacion o almacenamiento en la aplicacion.

**XII. Admin processes**: ejecutar tareas administrativas como procesos de un solo uso, no como hilos o subprocesos.

## Stateless por defecto

El principio mas critico para la orquestacion es la ausencia de estado local. Cuando un pod es destruido (por actualizacion, escalado o fallo), todo su sistema de archivos efimero se pierde. Si la aplicacion almacena sesiones, archivos temporales o datos en disco, esas pierdas se traducen en fallos de negocio. El estado debe externalizarse a servicios disenados para ello:

- **Sesiones**: Redis o Memcached en un Deployment separado.
- **Archivos**: S3, MinIO o un PersistentVolumeClaim gestionado por el orquestador.
- **Datos transaccionales**: bases de datos SQL/NoSQL con replicas externas.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
        - name: api
          image: mi-org/api:v1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: REDIS_URL
              value: "redis://redis-service:6379"
            - name: SESSION_STORE
              value: "redis"
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

> La combinacion de `preStop` hook y `sleep 10` permite que el pod reciba y complete las conexiones pendientes antes de ser eliminado, evitando errores 502 durante despliegues.

## Configuracion externa

Nunca se debe incluir configuracion sensible o de entorno en la imagen del contenedor. Las imagenes deben ser inmutables y ejecutables en cualquier entorno con los mismos binarios. La configuracion se inyecta a traves de:

- **ConfigMaps**: para configuracion no sensible (URLs, flags, archivos de configuracion).
- **Secrets**: para credenciales, certificados, tokens.
- **Variables de entorno**: para valores simples de configuracion.
- **Archivos montados**: para configuracion compleja que la aplicacion espere leer de un fichero.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  APP_LOG_LEVEL: "info"
  CACHE_TTL: "300"
  feature-flags.json: |
    {
      "new_checkout": true,
      "maintenance_mode": false
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
stringData:
  JWT_SECRET: "super-secret-token"
  DB_PASSWORD: "s3cur3p@ss"
```

```bash
# Crear un ConfigMap desde un fichero
kubectl create configmap api-config \
  --from-file=config.json \
  --from-literal=APP_LOG_LEVEL=info

# Crear un Secret desde litterales
kubectl create secret generic api-secrets \
  --from-literal=JWT_SECRET=super-secret-token \
  --from-literal=DB_PASSWORD=s3cur3p@ss
```

## Apagado limpio (Graceful Shutdown)

Cuando el orquestador decide detener un pod (por escalado, despliegue o mantencion), envia la senal `SIGTERM` al proceso principal y espera un periodo de gracia (por defecto 30 segundos) antes de enviar `SIGKILL`. Durante este intervalo, la aplicacion debe:

1. Dejar de aceptar conexiones nuevas (remover el endpoint de readiness).
2. Completar las solicitudes en curso.
3. Cerrar conexiones a bases de datos y colas.
4. Liberar recursos (archivos temporales, locks distribuidos).

```python
# Ejemplo en Python con FastAPI
import signal
import asyncio
from fastapi import FastAPI

app = FastAPI()
shutdown_event = asyncio.Event()

@app.on_event("startup")
async def startup():
    signal.signal(signal.SIGTERM, lambda s, f: asyncio.create_task(shutdown()))

async def shutdown():
    # Dejar de aceptar conexiones
    shutdown_event.set()
    # Cerrar conexiones pendientes
    await close_database_pool()
    await flush_pending_metrics()

@app.get("/health/ready")
async def readiness():
    if shutdown_event.is_set():
        from fastapi.responses import Response
        return Response(status_code=503)
    return {"status": "ready"}
```

> El endpoint de readiness debe devolver 503 inmediatamente tras recibir SIGTERM para que el kube-proxy elimine el pod del balanceo de carga. Esto evita que nuevas solicitudes lleguen a un pod que esta en proceso de apagado.

