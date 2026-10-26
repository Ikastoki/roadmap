# RBAC - Control de Acceso Basado en Roles

Kubernetes incorpora un sistema de autorización basado en roles que determina qué acciones puede realizar cada sujeto sobre los recursos del clúster. RBAC (Role-Based Access Control) opera sobre un modelo simple: se define un rol que describe los permisos, y luego se vincula a uno o más sujetos (usuarios, grupos o cuentas de servicio) mediante un binding. Este mecanismo reemplazó al modelo de control de acceso por defecto (ABAC), que resultaba difícil de gestionar en clústeres con múltiples equipos y servicios.

## Roles y ClusterRoles

Un `Role` define permisos sobre recursos dentro de un namespace específico. Un `ClusterRole` extiende esos permisos a nivel de clúster completo, incluyendo recursos a nivel de clúster como nodos o namespaces. Ambos tipos de roles especifican reglas que asocian verbos HTTP (get, list, watch, create, update, patch, delete) con recursos y subrecursos de la API de Kubernetes.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: produccion
  name: lector-pods
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list"]
    resourceNames: ["api-backend"]
```

> El campo `resourceNames` restringe los permisos a instancias específicas del recurso. En el ejemplo anterior, el rol solo permite acceder al deployment llamado `api-backend`, no a cualquier deployment del namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gestor-nodos
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["nodes/status"]
    verbs: ["patch"]
```

## RoleBindings y ClusterRoleBindings

Un binding conecta un rol con uno o más sujetos. Los sujetos pueden ser usuarios (identidades humanas o sistemas externos), grupos (agregaciones de identidades) o cuentas de servicio (identidades para procesos dentro del clúster). Un `RoleBinding` funciona dentro de un namespace, mientras que un `ClusterRoleBinding` aplica a todo el clúster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: binding-lector-pods
  namespace: produccion
subjects:
  - kind: User
    name: juan.garcia@empresa.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: devs-frontend
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: monitoring-agent
    namespace: produccion
roleRef:
  kind: Role
  name: lector-pods
  apiGroup: rbac.authorization.k8s.io
```

> Los usuarios en Kubernetes no son gestionados por el propio clúster. El API server depende de un mecanismo de autenticación externo (certificados, OIDC, tokens webhook) para validar la identidad. RBAC solo gestiona la autorización, no la autenticación.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: binding-admin-cluster
subjects:
  - kind: Group
    name: platform-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## Cuentas de servicio

Las cuentas de servicio proporcionan identidades a procesos que se ejecutan dentro del clúster, como controladores, agentes de monitoreo o pipelines de CI/CD. Cada namespace cuenta con una cuenta de servicio por defecto llamada `default`. Kubernetes monta automáticamente un token (o un projection de token) en cada pod que usa una cuenta de servicio, permitiendo al proceso autenticarse contra el API server.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-deploy
  namespace: produccion
automountServiceAccountToken: false
secrets:
  - name: pipeline-deploy-token
```

> El campo `automountServiceAccountToken: false` desactiva la inyección automática del token de la cuenta de servicio en los pods. Esto es una buena práctica de seguridad cuando el contenedor no necesita acceder al API server de Kubernetes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker-deploy
  namespace: produccion
spec:
  serviceAccountName: pipeline-deploy
  automountServiceAccountToken: true
  containers:
    - name: deployer
      image: kubectl-deployer:1.0
      env:
        - name: KUBERNETES_SERVICE_HOST
          value: "kubernetes.default.svc.cluster.local"
```

## Principio de minimo privilegio

El principio de mínimo privilegio exige que cada sujeto tenga solo los permisos necesarios para realizar su función, ni más ni menos. En la práctica, esto significa crear roles granulares que restrinjan el acceso a namespaces específicos, recursos específicos y verbos específicos. Una cuenta de servicio que solo necesita leer métricas no debe tener permisos de escritura; un desarrollador que despliega en un namespace de staging no debe poder modificar recursos en producción.

```bash
# Verificar los permisos de un sujeto
kubectl auth can-i get pods --namespace=produccion --as=juan.garcia@empresa.com
kubectl auth can-i create deployments --namespace=produccion --as=juan.garcia@empresa.com

# Listar todos los roles en un namespace
kubectl get roles -n produccion

# Listar los rolebindings en un namespace
kubectl get rolebindings -n produccion

# Ver los permisos detallados de un rol
kubectl describe role lector-pods -n produccion
```

> La herramienta `kubectl auth can-i` es fundamental para auditar permisos. Se puede usar con `--list` para obtener un resumen completo de todos los permisos que tiene un sujeto en un namespace determinado.

```bash
# Auditoria completa de permisos para un usuario
kubectl auth can-i --list --as=juan.garcia@empresa.com --namespace=produccion

# Verificar permisos a nivel de clúster
kubectl auth can-i get nodes --as=juan.garcia@empresa.com --as-group=platform-admins
```

## Agregacion de roles

Kubernetes permite agregar múltiples ClusterRoles en uno solo mediante etiquetas. Esto es útil para construir sistemas de permisos modulares donde un rol base puede ser extendido con permisos adicionales según el contexto del equipo o servicio.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoint
  labels:
    rbac.authorization.k8s.io/aggregate-to-monitoring: "true"
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.authorization.k8s.io/aggregate-to-monitoring: "true"
rules: []
```

> La agregación de roles se evalúa de forma dinámica. Cuando se crea, modifica o elimina un ClusterRole con la etiqueta de agregación, el ClusterRole compuesto se actualiza automáticamente. Esto permite que nuevos permisos se propaguen sin modificar la definición del rol principal.

