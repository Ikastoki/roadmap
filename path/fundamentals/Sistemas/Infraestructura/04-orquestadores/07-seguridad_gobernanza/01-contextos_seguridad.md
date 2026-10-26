# Contextos de Seguridad

La seguridad en un clúster de Kubernetes comienza por definir con precisión qué privilegios tiene cada contenedor dentro de su unidad de ejecución. Por defecto, los contenedores en un clúster estándar se ejecutan con permisos elevados: pueden acceder al sistema de archivos del nodo, modificar configuraciones de red e incluso escalar privilegios a nivel de host. Esto representa una superficie de ataque innecesariamente amplia. Los contextos de seguridad permiten restringir este comportamiento de forma declarativa, aplicando el principio de mínimo privilegio desde el punto de definición de la unidad de trabajo hasta el orquestador.

## Usuario no root

Ejecutar contenedores como el usuario raíz es una práctica habitual en entornos de desarrollo, pero constituye un riesgo significativo en producción. Un contenedor que se ejecuta como root puede explotar vulnerabilidades del kernel del nodo para escalar privilegios. Kubernetes ofrece la posibilidad de definir un usuario y grupo específicos mediante `runAsUser` y `runAsGroup`, además de `runAsNonRoot` para garantizar que el orquestador rechace cualquier intento de ejecución como root.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-seguro
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    fsGroup: 2000
  containers:
    - name: app
      image: nginx:1.25
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
```

> El campo `fsGroup`控制a ownership de los volúmenes montados en el contenedor. Todos los archivos creados dentro de un volumen tendrán este grupo como propietario, facilitando el acceso controlado entre contenedores que comparten volúmenes.

## Capabilities de Linux

El modelo de capabilities de Linux descompone los privilegios del superusuario en unidades granulares. En lugar de otorgar o denegar el acceso completo a root, se pueden habilitar o deshabilitar capabilities individuales como `NET_BIND_SERVICE` (para vincular puertos privilegiados), `SYS_TIME` (para modificar el reloj del sistema) o `NET_RAW` (para enviar paquetes de red sin procesar).

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
      - CHOWN
```

> La directiva `drop: ALL` elimina todas las capabilities heredadas, y luego `add` reintroduce únicamente las necesarias. Este patrón es la forma más segura de gestionar capabilities: partir de cero y añadir solo lo estrictamente requerido.

## Sistema de archivos de solo lectura

Un sistema de archivos de solo lectura impide que un contenedor escriba datos en su propio sistema de archivos raíz. Esto reduce el riesgo de que un atacante modifique binarios, scripts de inicio o archivos de configuración dentro del contenedor. Si la aplicación necesita escribir datos temporales, se pueden montar volúmenes `emptyDir` específicos para ese propósito.

```yaml
containers:
  - name: app
    image: nginx:1.25
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
      - name: tmp
        mountPath: /tmp
      - name: cache
        mountPath: /var/cache/nginx
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
```

## Escalada de privilegios

El campo `allowPrivilegeEscalation` controla si un proceso dentro del contenedor puede obtener más privilegios que su proceso padre. Esto incluye mecanismos como `setuid` binarios. Desactivar esta opción es una práctica recomendada que previene la escalada de privilegios a través de binarios del sistema que tienen el bit setuid activado.

```yaml
securityContext:
  allowPrivilegeEscalation: false
  privileged: false
  seccompProfile:
    type: RuntimeDefault
```

> El perfil `seccompProfile` con tipo `RuntimeDefault` aplica el perfil seccomp predeterminado del runtime del contenedor (como containerd o CRI-O), filtrando llamadas al sistema peligrosas. Para mayor restricción, se puede usar `Localhost` con un perfil personalizado.

## Contexto completo de seguridad

Un contexto de seguridad completo integra todas las restricciones mencionadas en un solo bloque, aplicable tanto a nivel de pod como a nivel de contenedor. Las configuraciones a nivel de pod se heredan por todos los contenedores, mientras que las configuraciones a nivel de contenedor pueden sobreescribir o complementar las del pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-seguro-completo
  labels:
    app: seguridad-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:1.25-alpine
      ports:
        - containerPort: 8080
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        privileged: false
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
    - name: run
      emptyDir: {}
```

> Para verificar que un contenedor se está ejecutando como el usuario esperado, se puede usar `kubectl exec <pod> -- id` y comprobar que el resultado muestra el UID definido en `runAsUser`.

## Politicas de seguridad a nivel de namespace

Para asegurar que todos los contenedores dentro de un namespace cumplan con los requisitos de seguridad, se pueden utilizar políticas de restricción. En lugar de definir contextos de seguridad individualmente por pod, se puede configurar un `PodSecurityStandards` o un `ConstraintTemplate` de OPA/Gatekeeper que valide automáticamente las configuraciones.

```bash
# Verificar que un pod cumple con las restricciones de seguridad
kubectl auth can-i create pods --as=system:serviceaccount:mynamespace:mysa

# Inspeccionar el securityContext de un pod en ejecución
kubectl get pod <nombre-pod> -o jsonpath='{.spec.containers[*].securityContext}'
```

