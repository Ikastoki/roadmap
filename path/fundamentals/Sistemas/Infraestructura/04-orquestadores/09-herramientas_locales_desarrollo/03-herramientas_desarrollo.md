# Herramientas de desarrollo

Desarrollar aplicaciones para un orquestador de contenedores presenta un desafio fundamental: el ciclo de codigo-build-push-deploy introduce latencia entre cada cambio que el desarrollador realiza y el momento en que puede observar su efecto en el entorno real. Las herramientas de desarrollo buscan comprimir este ciclo al minimo posible, idealmente a cero, permitiendo que el desarrollador trabaje con su editor y herramientas locales mientras la aplicacion se ejecuta dentro del cluster de Kubernetes. Este patron — mantener el entorno de ejecucion en el cluster pero el ciclo de desarrollo en la maquina local — requiere mecanismos sofisticados de sincronizacion de codigo, proxy inverso y tunneling de red que conecten ambos mundos de forma transparente.

## Skaffold

Skaffold, desarrollado por Google, implementa un pipeline completo de desarrollo continuo: detecta cambios en el codigo fuente, ejecuta las herramientas de build apropiadas (Docker, Jib, Bazel, Buildpacks), empuja las imagenes a un registry y despliega las manifests actualizadas en el cluster de Kubernetes. Todo esto se ejecuta de forma automatica en un loop, eliminando la necesidad de comandos manuales repetitivos.

### Instalacion

```bash
# Linux (binario)
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
chmod +x skaffold
sudo mv skaffold /usr/local/bin

# macOS (Homebrew)
brew install skaffold

# Verificar
skaffold version
```

### Configuracion basica

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta11
kind: Config
metadata:
  name: mi-aplicacion
build:
  artifacts:
    - image: mi-app
      context: .
      docker:
        dockerfile: Dockerfile
  local:
    push: false
    useBuildkit: true
deploy:
  kubectl:
    manifests:
      - k8s/*.yaml
profiles:
  - name: produccion
    build:
      local:
        push: true
      artifacts:
        - image: mi-app
          context: .
          docker:
            dockerfile: Dockerfile
    deploy:
      kubectl:
        manifests:
          - k8s/*.yaml
```

### Uso en desarrollo

```bash
# Iniciar ciclo de desarrollo automatico
# Reconstruye y redespliega en cada cambio detectado
skaffold dev

# Ejecutar una sola vez (build + deploy)
skaffold run

# Verificar estado del cluster
skaffold diagnose

# Ejecutar con un perfil especifico
skaffold dev --profile produccion

# Ejecutar tests integrados
skaffold test

# Generar manifests con variables
skaffold render

# Limpiar recursos desplegados
skaffold delete
```

### Port-forwarding automatico

```yaml
# En skaffold.yaml, seccion portForward
portForward:
  - resourceType: deployment
    resourceName: mi-api
    port: 8080
    localPort: 9090
  - resourceType: service
    resourceName: mi-frontend
    port: 3000
    localPort: 3000
```

> Skaffold mantiene un contexto de despliegue interno (similar a un "staging" en memoria) que rastrea exactamente que recursos estan desplegados actualmente. Esto permite una limpieza precisa cuando se detiene `skaffold dev`, eliminando solo los recursos que Skaffold ha creado, no todo el namespace.

## Telepresence

Telepresence aborda un problema diferente al de Skaffold: permite que un desarrollador ejecute su servicio individualmente en la maquina local mientras el resto del sistema se ejecuta en un cluster remoto. Telepresence crea un bidirectional VPN que intercepta el trafico destinado al servicio local dentro del cluster y lo redirige a la maquina del desarrollador, al tiempo que conecta el servicio local con el resto de servicios del cluster. Esto permite desarrollar y depurar una microservicio con las herramientas locales (debuggers, editores, base de datos local) sin desplegar el servicio completo en el cluster.

### Instalacion

```bash
# Linux / macOS
curl -fsSL https://app.gettelepresence.com/install.sh | sudo bash

# macOS (Homebrew)
brew install datawire/blackbird/telepresence-oss

# Verificar
telepresence version

# En macOS tambien se puede instalar via brew
brew install telepresence
```

### Conectar al cluster

```bash
# Conectar la maquina al cluster (requiere VPN-like tunnel)
telepresence connect

# Verificar conexion
telepresence status

# Desconectar
telepresence quit
```

### Intercept: redirigir trafico del cluster al local

```bash
# Interceptar todo el trafico de un servicio y redirigirlo a local
telepresence intercept mi-api --port 8080

# Interceptar solo unamespace especifico
telepresence intercept mi-api --namespace desarrollo --port 8080

# Ver intercepts activos
telepresence list

# Finalizar el intercept (restaura el trafico al cluster)
telepresence leave mi-api

# Desconectar completamente
telepresence quit
```

### Flujo tipico de desarrollo

```bash
# 1. Conectar al cluster
telepresence connect

# 2. Ejecutar el servicio en modo desarrollo local
python -m debugpy --listen 0.0.0.0:5678 --wait-for-client mi_api/main.py &

# 3. Interceptar el trafico del cluster
telepresence intercept mi-api --port 8080:5678

# 4. Ahora, cualquier request al servicio mi-api dentro del cluster
#    llega al debugger en la maquina local

# 5. Cuando termine de depurar
telepresence leave mi-api
telepresence quit
```

> Telepresence 2 (la version actual) utiliza agentes de sidecar en lugar de modification del deployment original, lo que lo hace menos invasivo y compatible con clusters que tienen politicas de seguridad estrictas. El agent se inyecta temporalmente y se elimina al finalizar el intercept.

## Dev Containers

Los contenedores de desarrollo (Dev Containers) llevan el concepto de entorno reproducible un paso alla del contenedor de ejecucion: encapsulan no solo la aplicacion sino todo el entorno de desarrollo — compiladores, linters, debuggers, extensiones del editor, configuraciones del shell — en un contenedor que se ejecuta en la maquina del desarrollador. VS Code con la extension Dev Containers y GitHub Codespaces son las implementaciones mas maduras de este patron.

### Definicion de Dev Container

```json
// .devcontainer/devcontainer.json
{
  "name": "Entorno de desarrollo mi-app",
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "hashicorp.terraform"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python"
      }
    }
  },
  "postCreateCommand": "pip install -r requirements.txt && pre-commit install",
  "forwardPorts": [8080, 5432],
  "runArgs": ["--network", "host"]
}
```

### Dockerfile del Dev Container

```dockerfile
# .devcontainer/Dockerfile
FROM mcr.microsoft.com/devcontainers/python:3.12

# Herramientas de desarrollo adicionales
RUN apt-get update && apt-get install -y \
    kubectl \
    helm \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Dependencias de Python
COPY requirements-dev.txt /tmp/requirements-dev.txt
RUN pip install -r /tmp/requirements-dev.txt

# Configuracion del shell
RUN echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

### Uso

```bash
# Abrir en Dev Container (desde VS Code)
# Cmd+Shift+P -> "Dev Containers: Reopen in Container"

# O ejecutar directamente con devcontainer CLI
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . bash
```

## Port-forwarding en Kubernetes

El port-forwarding es el mecanismo mas directo para acceder a un servicio del cluster desde la maquina del desarrollador. Kubernetes expone `kubectl port-forward` como herramienta nativa que establece un tunel TCP entre un pod o servicio del cluster y un puerto local.

### Uso basico

```bash
# Redirigir un puerto de un pod especifico
kubectl port-forward pod/mi-pod 8080:8080

# Redirigir a traves de un service (mas estable, ya que el service es abstraccion)
kubectl port-forward service/mi-service 8080:80

# Especificar namespace
kubectl port-forward -n produccion service/mi-api 9090:8080

# Escuchar en todas las interfaces (por defecto solo localhost)
kubectl port-forward --address 0.0.0.0 service/mi-service 8080:80
```

### Port-forward con multiplexado

```bash
# Usar un fichero kubeconfig especifico
KUBECONFIG=./mi-cluster.yaml kubectl port-forward service/mi-api 8080:80

# Mantener el tunel activo en background
nohup kubectl port-forward service/mi-api 8080:80 &

# Port-forward a multiples puertos simultaneamente
kubectl port-forward service/mi-stack 8080:80 5432:5432 6379:6379
```

### Consideraciones de seguridad

> `kubectl port-forward` solo esta disponible para usuarios con permisos `pods/portforward` en el RBAC del cluster. En entornos de produccion, este permiso debe restringirse estrictamente ya que expone servicios internos del cluster directamente a la maquina del desarrollador, potencialmente saltando politicas de red (NetworkPolicy) y autenticacion.

## Patron de desarrollo: bridge entre local y remoto

La combinacion de estas herramientas permite establecer un patron de desarrollo que maximiza la velocidad de iteracion sin sacrificar la fidelidad del entorno:

```
┌─────────────────────────────────────────────────┐
│              Maquina local                       │
│                                                  │
│  ┌────────────┐    ┌────────────┐                │
│  │   Editor   │───▶│ Skaffold   │───┐            │
│  │  (VSCode)  │    │  (watch)   │   │            │
│  └────────────┘    └────────────┘   │            │
│                                     ▼            │
│  ┌────────────┐    ┌────────────────────────┐   │
│  │ Debugger   │◀───│ Telepresence Intercept │   │
│  │ (local)    │    │  (bidirectional VPN)   │   │
│  └────────────┘    └────────────────────────┘   │
└─────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────┐
│          Cluster Kubernetes (remoto)             │
│                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │ Servicio │ │ Servicio │ │ Servicio │        │
│  │    A     │ │    B     │ │    C     │  ...   │
│  └──────────┘ └──────────┘ └──────────┘        │
│                                                  │
│  El trafico de "mi-servicio" se redirige         │
│  a la maquina local via Telepresence             │
└─────────────────────────────────────────────────┘
```

Para servicios que se encuentran en fase de estabilizacion, Skaffold gestiona el ciclo completo de build y deploy. Para microservicios individuales en desarrollo activo, Telepresence permite iterar con la velocidad del entorno local mientras se valida la integracion con el sistema completo desplegado en el cluster. Los Dev Containers garantizan que todos los desarrolladores del equipo operen con las mismas herramientas y versiones, eliminando la classe de bugs que surgen de diferencias entre entornos.

