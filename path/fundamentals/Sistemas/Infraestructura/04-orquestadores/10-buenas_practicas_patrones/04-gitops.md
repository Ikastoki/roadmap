# GitOps

GitOps es un paradigma de operaciones donde el repositorio Git se convierte en la unica fuente de verdad (single source of truth) para el estado declarativo de la infraestructura y las aplicaciones. Todo lo que se ejecuta en el cluster debe existir primero como un commit en Git. No hay excepciones: no se aplican manifests directamente con `kubectl apply`, no se hacen cambios manuales en produccion, no se ejecutan scripts de despliegue ad-hoc. El flujo unidireccional es: un desarrollador modifica el repositorio, un agente detecta el cambio, y el cluster se sincroniza automaticamente para reflejar el nuevo estado declarado en Git.

Este modelo transforma la operacion de Kubernetes de un proceso imperativo ("ejecuta estos comandos") a uno declarativo ("este es el estado deseado, hazlo realidad"). Las consecuencias son profundas: cada cambio queda registrado con autor, timestamp y justificacion (via commit message o PR description), los despliegues se revierten con `git revert`, y los conflictos se resuelven con los flujos de Pull Request que ya son familiares para los equipos de desarrollo.

## Principios fundamentales

GitOps no es solo usar Git para almacenar manifests; es un conjunto de principios operativos que transforman como un equipo interactua con sus sistemas:

1. **Declarativismo**: el estado completo del sistema debe ser expresable de forma declarativa. Kubernetes es inherentemente declarativo, lo que lo hace idoneo para GitOps.

2. **Versionado inmutable**: el estado deseado se almacena en commits inmutables de Git. Cada commit es un snapshot verificable y auditado del sistema en un momento dado.

3. **Automatizacion de aplicacion**: un agente automatico detecta las diferencias entre el estado declarado en Git y el estado real del cluster, y aplica los cambios necesarios para converger al estado deseado.

4. **Reconciliacion continua**: el agente no solo aplica cambios una vez, sino que continua verificando que el estado real coincide con el declarado. Si alguien modifica manualmente un recurso en el cluster (drift), el agente lo revierte automaticamente.

## Flux

Flux es un conjunto de controladores de Kubernetes que implementan GitOps de forma nativa dentro del cluster. Flux se ejecuta como pods dentro del propio cluster y observa los repositorios configurados, detectando cambios y aplicandolos automaticamente. Su arquitectura modular permite componer diferentes fuentes (Git, Helm, OCI) con diferentes controladores de reconciliacion.

```yaml
# Instalacion de Flux con Bootstrap
# Este comando instala los CRDs y controladores de Flux en el cluster
flux bootstrap github \
  --owner=mi-org \
  --repository=gitops-repo \
  --branch=main \
  --path=./clusters/production \
  --personal
```

```yaml
# GitRepository: define la fuente del estado declarativo
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: mi-aplicacion
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/mi-org/gitops-repo
  ref:
    branch: main
  secretRef:
    name: git-credentials
---
# Kustomization: define como aplicar los manifests del repositorio
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mi-aplicacion
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/produccion
  prune: true
  sourceRef:
    kind: GitRepository
    name: mi-aplicacion
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: backend
      namespace: produccion
  timeout: 3m
---
# HelmRelease: gestion de releases de Helm via GitOps
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: mi-aplicacion
  namespace: flux-system
spec:
  interval: 10m
  chart:
    spec:
      chart: mi-aplicacion
      version: "1.2.x"
      sourceRef:
        kind: HelmRepository
        name: mi-charts
  values:
    replicaCount: 3
    image:
      repository: mi-org/backend
      tag: "1.2.0"
  valuesFrom:
    - kind: ConfigMap
      name: app-values
      valuesKey: values.yaml
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
  rollback:
    cleanupOnFail: true
```

```bash
# Verificar el estado de sincronizacion
flux get kustomizations

# Forzar una reconciliacion manual
flux reconcile kustomization mi-aplicacion --with-source

# Ver los eventos de reconciliacion
flux logs --kind=kustomization --name=mi-aplicacion

# Suspender la sincronizacion (util en emergencias)
flux suspend kustomization mi-aplicacion

# Reanudar la sincronizacion
flux resume kustomization mi-aplicacion
```

## ArgoCD

ArgoCD es una herramienta de GitOps que proporciona una interfaz grafica rica ademas de su CLI. A diferencia de Flux, que es puramente operativo, ArgoCD esta disenado para equipos que necesitan visibilidad visual del estado de sus despliegues. Implementa el patron de controlador de reconciliacion: un proceso observa el estado declarado en Git y lo compara con el estado real del cluster, ejecutando los comandos necesarios para converger.

```yaml
# Application: define una aplicacion que ArgoCD gestiona
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-aplicacion
  namespace: argocd
  # Evitar que ArgoCD gestione su propio recurso de Application
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/mi-org/gitops-repo.git
    targetRevision: main
    path: apps/produccion
    # Opcion: usar Kustomize directamente
    kustomize:
      images:
        - mi-org/backend:1.2.0
  destination:
    server: https://kubernetes.default.svc
    namespace: produccion
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  # Revision history para rollback visual
  revisionHistoryLimit: 10
```

```yaml
# AppProject: agrupa aplicaciones con restricciones
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: produccion
  namespace: argocd
spec:
  description: Proyecto de aplicaciones en produccion
  sourceRepos:
    - "https://github.com/mi-org/gitops-repo.git"
    - "https://charts.bitnami.com/bitnami"
  destinations:
    - namespace: produccion
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: ""
      kind: "*"
    - group: apps
      kind: "*"
    - group: networking.k8s.io
      kind: "*"
  roles:
    - name: developer
      description: Acceso de desarrollo
      policies:
        - p, proj:produccion:developer, applications, get, produccion/*, allow
        - p, proj:produccion:developer, applications, sync, produccion/*, allow
```

```bash
# Instalar ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Obtener la contrasena inicial
argocd admin initial-password -n argocd

# Sincronizar una aplicacion
argocd app sync mi-aplicacion

# Ver el historial de despliegues
argocd app history mi-aplicacion

# Revertir a una revision anterior
argocd app rollback mi-aplicacion 5

# Verificar diferencias entre Git y cluster
argocd app diff mi-aplicacion
```

> ArgoCD y Flux no son mutuamente excluyentes. Equipos grandes a menudo usan ArgoCD para la interfaz visual y la gestion de permisos, mientras que Flux gestiona los componentes criticos del cluster (monitoring, logging, networking) como controladores internos.

## Flujo completo de un despliegue GitOps

El ciclo de vida de un cambio en un entorno GitOps sigue un flujo predecible:

1. Un desarrollador crea un branch, modifica los manifests o actualiza una version de imagen en el repositorio Git.
2. Abre un Pull Request que es revisado por el equipo.
3. Tras la aprobacion y merge a `main`, el agente de GitOps (Flux o ArgoCD) detecta el cambio.
4. El agente compara el nuevo estado declarado con el estado actual del cluster.
5. Genera un plan de cambios y los aplica de forma ordenada (namespaces primero, luego Deployments, luego Services).
6. Espera a que los Health Checks confirmen que los pods estan listos.
7. Actualiza el estado de sincronizacion y notifica (opcionalmente via Slack, webhook, etc.).

```bash
# Verificar el estado de sincronizacion con ArgoCD
argocd app get mi-aplicacion --hard-refresh

# Verificar con Flux
flux get kustomizations mi-aplicacion
flux get helmreleases -A
```

## Beneficios tangibles

La adopcion de GitOps proporciona beneficios concretos que van mas alla de la comodidad operativa. La trazabilidad completa de cada cambio (quien, cuando, por que) facilita la auditoria y el cumplimiento normativo. El rollback se reduce a un `git revert` o un comando del orquestador, eliminando la necesidad de scripts de emergencia. La colaboracion basada en Pull Requests introduce revision de pares automatica en cada cambio de infraestructura, reduciendo significativamente la probabilidad de errores en produccion.

> GitOps no elimina la necesidad de monitoreo ni de respuesta a incidentes. Complementa estas practicas proporcionando un mecanismo de recuperacion rapida y consistente: si el estado del cluster diverge por cualquier razon, la reconciliacion automatica lo restaura al estado declarado en Git.

