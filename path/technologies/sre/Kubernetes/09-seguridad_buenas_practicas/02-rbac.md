# RBAC

Kubernetes carecia de un sistema de autenticacion y autorizacion granular en sus primeras versiones. Cualquier persona con acceso al API Server tenia control total sobre el cluster. Con la adopcion empresarial, se hizo evidente la necesidad de un modelo de permisos que permitiera a diferentes equipos convivir en el mismo cluster sin pisarse ni exponer recursos sensibles. RBAC (Role-Based Access Control) se convirtio en el mecanismo estandar de autorizacion en Kubernetes, permitiendo definir que acciones puede realizar cada identidad sobre cada recurso.

## Roles y ClusterRoles

Un `Role` define un conjunto de permisos dentro de un namespace concreto. No puede otorgar acceso a recursos fuera de ese namespace ni a recursos que no pertenezcan a ningun namespace (como nodos o namespaces).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: desarrollo
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Un `ClusterRole` es identico en estructura pero no esta ligado a un namespace. Sirve para conceder permisos sobre recursos globales (nodos, namespaces, PersistentVolumes) o para ser reutilizado en multiples namespaces mediante bindings.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-monitor
rules:
  - apiGroups: [""]
    resources: ["nodes", "namespaces", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]
```

La estructura de `rules` sigue siempre el mismo patron: `apiGroups`, `resources` y `verbs`. Los verbs disponibles son `get`, `list`, `watch`, `create`, `update`, `patch` y `delete`. El verbo `*` concede todos los permisos sobre el recurso.

> Un ClusterRole puede asociarse a usuarios dentro de un namespace especifico usando un RoleBinding. Esto permite definir roles globales reutilizables pero aplicarlos solo a namespaces concretos, combinando la flexibilidad del ClusterRole con el aislamiento del RoleBinding.

## RoleBindings y ClusterRoleBindings

Un `RoleBinding` asigna un Role o ClusterRole a un conjunto de sujetos (usuarios, grupos, ServiceAccounts) dentro de un namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: leer-pods
  namespace: desarrollo
subjects:
  - kind: User
    name: juan@empresa.com
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: pipeline-ci
    namespace: desarrollo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Un `ClusterRoleBinding` funciona igual pero asigna permisos de forma global, afectando a todos los namespaces. Solo puede hacer referencia a un ClusterRole, no a un Role, ya que el Role esta limitado a un namespace y un ClusterRoleBinding tiene alcance global.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admins
subjects:
  - kind: Group
    name: sre-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

El `roleRef` es inmutable una vez creado el binding. Para cambiar el rol asociado a un conjunto de sujetos, es necesario eliminar y recrear el binding. Esta restriccion evita modificaciones accidentales que podrian otorgar permisos no deseados.

## Service Accounts

Las Service Accounts son identidades pensadas para procesos dentro del cluster, no para personas. Cada Pod se asocia a una Service Account, y el token de esa cuenta se monta automaticamente en `/var/run/secrets/kubernetes.io/serviceaccount/token`, permitiendo al Pod autenticarse contra el API Server.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: automations-sa
  namespace: produccion
---
apiVersion: v1
kind: Pod
metadata:
  name: automation-pod
  namespace: produccion
spec:
  serviceAccountName: automations-sa
  containers:
    - name: automation
      image: automation-app
```

Cada namespace tiene una Service Account por defecto llamada `default`. Si un Pod no especifica `serviceAccountName`, se le asigna la default. La practica recomendada es no otorgar permisos a la Service Account default de ningun namespace para evitar que Pods olviden configurar su identidad y terminen con permisos no intencionados.

```bash
# Verificar que la Service Account default no tiene permisos
kubectl auth can-i --list --as=system:serviceaccount:default:default

# Deshabilitar montaje automatico del token para la default
kubectl patch serviceaccount default -p '{"automountServiceAccountToken": false}'
```

## Buenas practicas con RBAC

El principio de minimo privilegio aplica directamente a RBAC: cada identidad debe tener exactamente los permisos que necesita, ni uno mas. Conceder `cluster-admin` a un desarrollador que solo necesita ver logs de sus Pods es una vulnerabilidad evitable.

```yaml
# En lugar de cluster-admin, un rol especifico
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: desarrollo
  name: logs-viewer
rules:
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

El uso de `apiGroups` y `resources` debe ser lo mas especifico posible. Usar comodines como `apiGroups: ["*"]` o `resources: ["*"]` solo cuando sea estrictamente necesario. Las reglas se evaluan en union: si un RoleBinding otorga `get` en `pods` y otro binding otorga `delete` en `pods`, el sujeto tendra ambos permisos.

Es importante auditar periodicamente los permisos del cluster. Herramientas como `kubectl auth can-i` permiten verificar que permisos tiene un usuario o Service Account:

```bash
kubectl auth can-i get pods --as=juan@empresa.com --namespace=produccion
kubectl auth can-i delete deployments --as=system:serviceaccount:ci:pipeline
```

Los grupos de usuarios son una forma eficaz de gestionar permisos a escala. En lugar de crear bindings individuales para cada persona, se asignan roles a grupos (LDAP, OIDC) y se gestionan las membresias fuera de Kubernetes:

```yaml
subjects:
  - kind: Group
    name: "developers"
    apiGroup: rbac.authorization.k8s.io
```

Los `AggregationRule` en ClusterRoles permiten combinar multiples roles en uno solo. Si un ClusterRole A agrega los roles B y C, cualquier cambio en B o C se refleja automaticamente en A, facilitando la gestion permisos compuestos:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-full-access
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.aggregation/developer: "true"
rules: []
```

> La inmutabilidad del `roleRef` en los bindings es una caracteristica de seguridad, no una limitacion. Si pudieras modificar el `roleRef`, un atacante que comprometiera un RoleBinding podria escalar privilegios sin crear nuevos recursos que pudieran ser detectados por herramientas de auditoria.

