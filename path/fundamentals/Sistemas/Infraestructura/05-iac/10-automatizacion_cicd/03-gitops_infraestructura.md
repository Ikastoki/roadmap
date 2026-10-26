# GitOps para infraestructura

GitOps extiende el modelo de declaracion de IaC con un principio fundamental: el repositorio Git no es solo un almacen de configuraciones, sino la fuente unica de verdad sobre el estado deseado de toda la infraestructura. Un operador desplegado en el clúster o en la nube observa continuamente el repositorio, compara el estado declarado en los archivos con el estado real de los recursos y ejecuta las acciones necesarias para reconciliar ambas diferencias. Esto invierte el modelo tradicional de despliegue: en lugar de que un pipeline empuje cambios al entorno, el entorno jala los cambios desde Git.

La diferencia con un pipeline CI/CD clasico es sustancial. En el modelo de push, un pipeline ejecuta `terraform apply` y modifica el entorno directamente. En GitOps, un operador residente en el entorno detecta que el estado real diverge del estado declarado en Git y ejecuta la correccion automaticamente. Esto significa que si alguien modifica manualmente un recurso en la consola de AWS o Kubernetes, el operador lo detecta y revierte el cambio, manteniendo la consistencia sin intervencion humana.

## Crossplane

Crossplane es un controlador de Kubernetes que extiende el concepto de GitOps a la infraestructura cloud. Permite definir recursos de AWS, GCP o Azure como si fueran recursos nativos de Kubernetes, utilizando CRDs (Custom Resource Definitions) y reconciliandolos continuamente.

Un `Claim` en Crossplane es la abstraction que un desarrollador consume: declara que necesita una base de datos sin conocer los detalles de implementacion especificos de cada proveedor cloud. El `CompositeResourceDefinition` (XRD) define el contrato y el `Composition` contiene la logica de como traducir ese claim a recursos concretos del proveedor.

```yaml
# database-claim.yaml - El desarrollador declara lo que necesita
apiVersion: database.example.com/v1alpha1
kind: PostgreSQL
metadata:
  name: api-database
  namespace: produccion
spec:
  parameters:
    storageGB: 100
    version: "15"
    highAvailability: true
    backupRetentionDays: 7
  compositionSelector:
    matchLabels:
      provider: aws
  writeConnectionSecretToRef:
    name: api-database-credentials
    namespace: produccion
```

```yaml
# xrd.yaml - Definicion del recurso compuesto
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: postgresqls.database.example.com
spec:
  group: database.example.com
  names:
    kind: PostgreSQL
    plural: postgresqls
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:
                      type: integer
                      minimum: 20
                      maximum: 1000
                    version:
                      type: string
                      enum: ["13", "14", "15"]
                    highAvailability:
                      type: boolean
                    backupRetentionDays:
                      type: integer
                      minimum: 1
                      maximum: 35
                  required:
                    - storageGB
                    - version
```

```yaml
# composition.yaml - Como se traduce el claim a recursos de AWS
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql-aws
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: database.example.com/v1alpha1
    kind: PostgreSQL
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.crossplane.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            dbInstanceClass: db.r6g.large
            engine: postgres
            masterUsername: admin
            storageType: gp3
            storageEncrypted: true
            deletionProtection: true
            enablePerformanceInsights: true
            tags:
              - key: managed-by
                value: crossplane
          providerConfigRef:
            name: aws-production
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.version
          toFieldPath: spec.forProvider.engineVersion
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.highAvailability
          toFieldPath: spec.forProvider.multiAZ
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: metadata.name
              - fromFieldPath: metadata.namespace
            strategy: string
            string:
              fmt: "%s-%s-credentials"
          toFieldPath: spec.forProvider.masterUserPasswordSecretRef.name
```

Crossplane reconcilia el ciclo de vida completo del recurso. Si el claim solicita 100 GB de almacenamiento y el estado real tiene 50 GB, Crossplane ejecuta un `Modify` sobre la instancia RDS. Si alguien elimina la instancia RDS desde la consola de AWS, Crossplane la recrea automaticamente para que coincida con el estado declarado en Git.

> Crossplane requiere que los CRDs esten instalados en un clúster Kubernetes funcional. Esto introduce una dependencia: la infraestructura cloud depende de un clúster que, a su vez, podria estar definido por la propia plataforma de Crossplane. Este fenomeno se conoce como "bootstrap problem" y se resuelve tipicamente desplegando un clúster minimo inicial manualmente o con Terraform, y delegando despues a Crossplane la gestion de todo el resto.

## ArgoCD

ArgoCD se centra exclusivamente en la sincronizacion de Kubernetes, pero en un escenario de infraestructura hibrida puede utilizarse para gestionar tanto aplicaciones como la capa de IaC que se despliega como Helm charts o Kustomize overlays.

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infraestructura-core
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: infraestructura
  source:
    repoURL: https://github.com/mi-empresa/infra-repo.git
    targetRevision: main
    path: k8s/platform
    kustomize:
      namePrefix: prod-
      images:
        - ghcr.io/mi-empresa/crossplane-provider-aws:1.4.0
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
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
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: ""
      kind: Secret
      jsonPointers:
        - /data
```

La directiva `selfHeal: true` es la que convierte ArgoCD en un verdadero sistema GitOps: si el estado real del clúster diverge de lo que esta declarado en Git (por ejemplo, alguien ejecuta un `kubectl delete` manual sobre un recurso), ArgoCD detecta la diferencia y restaura el estado original. `prune: true` garantiza que los recursos eliminados del repositorio se eliminen del clúster, manteniendo el repositorio como la autoridad definitiva.

```yaml
# AppProject - Delimita que puede desplegar ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infraestructura
  namespace: argocd
spec:
  description: Proyecto de infraestructura core
  sourceRepos:
    - 'https://github.com/mi-empresa/infra-repo.git'
  destinations:
    - namespace: crossplane-system
      server: https://kubernetes.default.svc
      name: '*'
    - namespace: argocd
      server: https://kubernetes.default.svc
      name: '*'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  orphanedResources:
    warn: true
```

## Reconciliacion continua

El concepto central que conecta Crossplane y ArgoCD es el ciclo de reconciliacion: un proceso observador que ejecuta continuamente la secuencia observar -> comparar -> actuar. Este ciclo no se ejecuta una vez por despliegue, sino que esta permanentemente activo, verificando cada pocos segundos que el estado real coincide con el estado declarado.

```
Git (estado deseado) -> Operador -> Estado real (cloud/K8s)
       |                                |
       +-- diff -- reconciliacion -------+
```

En Terraform clasico, la reconciliacion solo ocurre cuando un humano ejecuta `terraform plan` o cuando un pipeline se dispara por un cambio en el repositorio. En GitOps, la reconciliacion es continua y automatica, lo que significa que las desviaciones no autorizadas (configuration drift) se corrigen sin necesidad de que nadie las detecte manualmente.

> La reconciliacion continua tiene implicaciones sobre los costes: cada ciclo de Crossplane o ArgoCD puede ejecutar llamadas a APIs de cloud (DescribeInstances, ListBuckets, etc.). En entornos con miles de recursos, es fundamental configurar intervalos de reconciliacion razonables y utilizar `providerConfig` con politicas de rate limiting para evitar sobrecargar las APIs del proveedor.

