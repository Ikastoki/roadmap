# Politicas de Seguridad

Kubernetes no impone restricciones de seguridad por defecto: un usuario con acceso al API server puede crear pods privilegiados, montar volúmenes sensibles o ejecutar contenedores como root. Las politicas de seguridad cierran esta brecha mediante mecanismos de admisión que validan o mutan los recursos antes de que se almacenen en el estado del clúster. Estas politicas se aplican en la fase de admisión del pipeline de creación de recursos, justo después de la autenticación y autorización, pero antes de que el objeto persista en etcd.

## Pod Security Standards

Los Pod Security Standards reemplazan al descontinuado PodSecurityPolicy. Establecen tres niveles de restricción que se aplican a namespaces mediante etiquetas. Cada nivel define qué campos de seguridad están permitidos, denegados o restringidos en las unidades de trabajo.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: produccion
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

> Los tres modos de aplicación son `enforce` (rechaza pods que no cumplan), `audit` (registra eventos de auditoría) y `warn` (devuelve advertencias al cliente). Es recomendable usar los tres en modo restrictivo, pero con `enforce` en `warn` inicialmente para identificar pods existentes que violen la politica sin bloquear su despliegue.

### Nivel Privileged

El nivel `privileged` no impone ninguna restricción. Permite contenedores privilegiados, montaje de volúmenes del sistema de archivos del host, y todas las capabilities de Linux. Este nivel está destinado a workloads del sistema operativo y componentes de infraestructura como redes o almacenamiento que requieren acceso directo al hardware.

### Nivel Baseline

El nivel `baseline` prohíbe las configuraciones más peligrosas sin requerir cambios significativos en las aplicaciones. Impide contenedores privilegiados, el montaje de ciertos volúmenes sensibles (como `/proc` o `/sys` del host), y el uso de capabilities peligrosas como `SYS_ADMIN` o `NET_RAW`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-baseline
  namespace: staging
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:1.25
      securityContext:
        allowPrivilegeEscalation: false
```

### Nivel Restricted

El nivel `restricted` es el más estricto y está diseñado para workloads de aplicaciones estándar. Requiere que los contenedores se ejecuten como no root, desactiven la escalada de privilegios, eliminen todas las capabilities y utilicen un perfil seccomp. Este nivel es el recomendado para la mayoría de las aplicaciones en producción.

## Admission Controllers

Los admission controllers son componentes del API server que interceptan las solicitudes de creación o modificación de recursos. Existen dos tipos: los de validación, que rechazan solicitudes que no cumplan ciertas condiciones, y los de mutación, que modifican los objetos antes de almacenarlos. Kubernetes incluye varios admission controllers habilitados por defecto, y permite la instalación de controladores personalizados mediante webhooks.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validador-imagenes
webhooks:
  - name: validar-imagenes.empresa.com
    clientConfig:
      service:
        name: webhook-server
        namespace: kube-system
        path: /validar-imagenes
      caBundle: <base64-ca-cert>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
    failurePolicy: Fail
    admissionReviewVersions: ["v1"]
```

> El campo `failurePolicy` determina qué ocurre cuando el webhook no responde. `Fail` rechaza la solicitud (modo seguro), mientras que `Ignore` permite que la solicitud se procese sin validación (modo tolerante). En producción, `Fail` es la opción recomendada por defecto.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: inyectar-seguridad
webhooks:
  - name: inyectar-contexto.empresa.com
    clientConfig:
      service:
        name: webhook-server
        namespace: kube-system
        path: /inyectar-contexto
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    mutationPatch: |
      [
        {
          "op": "add",
          "path": "/spec/containers/0/securityContext/allowPrivilegeEscalation",
          "value": false
        },
        {
          "op": "add",
          "path": "/spec/containers/0/securityContext/readOnlyRootFilesystem",
          "value": true
        }
      ]
```

## OPA/Gatekeeper

Open Policy Agent (OPA) es un motor de politicas general que se integra con Kubernetes mediante Gatekeeper. Permite definir politicas personalizadas usando el lenguaje Rego y aplicarlas como constraint templates. Gatekeeper opera como un admission controller que valida cada solicitud de creación o modificación de recursos contra las politicas definidas.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoimagentaglatest
spec:
  crd:
    spec:
      names:
        kind: K8sNoImageTagLatest
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoimagentaglatest

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          endswith(container.image, ":latest")
          msg := sprintf("El contenedor '%v' usa la etiqueta 'latest'. Usa una etiqueta específica para la imagen.", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, ":")
          msg := sprintf("El contenedor '%v' no tiene etiqueta de versión definida. Especifica una etiqueta explícita.", [container.name])
        }
```

> Las constraint templates son reutilizables: una sola template puede ser instanciada múltiples veces con diferentes parámetros. Por ejemplo, una template que restringe imágenes puede aplicarse con diferentes listas de registros permitidos por namespace.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoImageTagLatest
metadata:
  name: prohibir-tag-latest
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - produccion
      - staging
  parameters:
    allowedRegistries:
      - "registry.empresa.com/"
      - "gcr.io/proyecto-empresa/"
```

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sregistriopermitido
spec:
  crd:
    spec:
      names:
        kind: K8sRegistroPermitido
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedRegistries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sregistriopermitido

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not startswith_any(image, input.parameters.allowedRegistries)
          msg := sprintf("La imagen '%v' del contenedor '%v' no proviene de un registro permitido. Registros permitidos: %v", [image, container.name, input.parameters.allowedRegistries])
        }

        startswith_any(str, prefixes) {
          prefix := prefixes[_]
          startswith(str, prefix)
        }
```

> OPA/Gatekeeper incluye un conjunto de constraint templates predefinidas en el proyecto `library` de GitHub. Estas cubren casos comunes como la exigencia de etiquetas, la restricción de imágenes de Docker Hub y la validación de limits de recursos.

