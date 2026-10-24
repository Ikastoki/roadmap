# Pod Security Standards

Kubernetes no nacio con un sistema de politicas de seguridad integrado. Durante anos, la unica forma de restringir lo que un Pod podia hacer era mediante controladores externos como PodSecurityPolicy (PSP), una API compleja que resultado dificil de configurar y mantener. PSP fue deprecada en Kubernetes 1.21 y eliminada en 1.25, reemplazada por los Pod Security Standards (PSS) y el Pod Security Admission (PSA), un enfoque mas simple y mantenible integrado directamente en el API Server.

## Los tres niveles de seguridad

Los Pod Security Standards definen tres niveles de restriccion que cubren desde entornos permisivos hasta configuraciones endurecidas. Cada nivel es un perfil que agrupa una serie de restricciones sobre SecurityContext, capacidades, volumenes, y otros aspectos del Pod.

**Privileged:** el nivel mas Permisivo. No impone ninguna restriccion. Los Pods pueden ejecutarse como root, usar cualquier capacidad, montar cualquier tipo de volumen y acceder a dispositivos del host. Es el nivel adecuado para componentes de infraestructura del cluster que necesitan acceso privilegiado: controladores CNI, agentes de monitorizacion, herramientas de backup.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
    - name: cni-agent
      image: cni-agent:latest
      securityContext:
        privileged: true
        capabilities:
          add:
            - ALL
  hostNetwork: true
  hostPID: true
```

**Baseline:** un nivel intermedio que bloquea las configuraciones claramente inseguras pero permite la mayoria de las aplicaciones convencionales sin modificaciones. Prohibe contenedores privilegiados, hostPID, hostNetwork, hostIPC, y capacidades peligrosas como `SYS_ADMIN`. Es el perfil por defecto recomendado para namespaces de proposito general.

```yaml
# Lo que Baseline permite:
securityContext:
  runAsNonRoot: false        # Permite root
  capabilities:
    drop:
      - ALL                  # Exigible si se requiere, pero permite anadir NET_BIND_SERVICE
    add:
      - NET_BIND_SERVICE

# Lo que Baseline prohibe:
# - privileged: true
# - hostPID: true, hostNetwork: true, hostIPC: true
# - capabilities como SYS_ADMIN, SYS_PTRACE, SYS_MODULE
# - tipos de volumen hostPath inseguros
```

**Restricted:** el nivel mas estricto. Disenado siguiendo las mejores practicas de hardening de contenedores. Exige `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, eliminacion de todas las capacidades, y prohibicion de cualquier tipo de acceso al host. Es el nivel recomendado para aplicaciones de produccion que no requieren privilegios especiales.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        readOnlyRootFilesystem: true
        runAsNonRoot: true
```

## Aplicacion de politicas por namespace

El Pod Security Admission se configura mediante etiquetas en el namespace. No requiere instalar admission controllers externos ni operadores. El API Server evalua los Pods contra el nivel especificado y actua segun el modo configurado.

```bash
kubectl create namespace produccion
kubectl label namespace produccion \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

Cada etiqueta define un modo distinto:

- **enforce:** el API Server rechaza los Pods que no cumplen con el nivel especificado. Es el modo mas restrictivo, ideal para namespaces de produccion.
- **audit:** los Pods que violan el nivel se registran en los audit logs pero no se bloquean. Util para evaluar el impacto de endurecer un namespace antes de activar enforce.
- **warn:** los Pods que violan el nivel generan una advertencia al usuario al crear o actualizar el recurso, pero no se bloquean. Combina bien con audit para dar visibilidad a los desarrolladores.

```bash
# Verificar el nivel de seguridad aplicado a un namespace
kubectl label --list namespace produccion
# Ver Pods que violan la politica
kubectl get pods -n produccion -o yaml | grep "pod-security.kubernetes.io"
```

> Los tres modos pueden combinarse con niveles distintos. Un namespace tipico tiene `enforce=restricted`, `audit=baseline` para detectar Pods que necesiten relajacion, y `warn=baseline` para notificar a los usuarios. Esta combinacion permite una migracion gradual hacia Restricted sin romper cargas de trabajo existentes.

## Admission controllers y validacion

El Pod Security Admission es un admission controller integrado que se activa por defecto en Kubernetes 1.23+. Cuando un usuario envia un manifiesto de Pod al API Server, el admission controller intercepta la solicitud y la evalua contra las etiquetas PSS del namespace.

El flujo de validacion es el siguiente:

1. El usuario envia un Pod (o un controlador como Deployment que crea Pods).
2. El API Server ejecuta los admission controllers de validacion.
3. PodSecurity evalua el Pod contra el nivel `enforce` del namespace.
4. Si el Pod cumple, se persiste en etcd y se devuelve una respuesta exitosa.
5. Si el Pod no cumple, se rechaza con un mensaje que detalla que restricciones se violaron.

```bash
# Intento de crear un Pod privilegiado en un namespace restricted
kubectl apply -n produccion -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        privileged: true
EOF
# Error: Pod "test-privileged" is forbidden: violates PodSecurity "restricted:latest":
#   privileged container (container "nginx" must not set securityContext.privileged=true)
```

## Exclusiones y excepciones

Algunos Pods no pueden cumplir con ciertos niveles por razones tecnicas legitimas. Los componentes del sistema (kube-system, kube-public) suelen tener exenciones automaticas. Para namespaces de usuario, se pueden crear excepciones mediante `PodSecurityStandard` exemptions en la configuracion del API Server o utilizando `SecurityContextConstraints` en entornos OpenShift.

El enfoque recomendado es tener namespaces con distintos niveles de seguridad segun la criticidad:

```bash
# Namespace para componentes de infraestructura
kubectl label namespace infra pod-security.kubernetes.io/enforce=privileged

# Namespace para aplicaciones normales
kubectl label namespace apps pod-security.kubernetes.io/enforce=baseline

# Namespace para aplicaciones criticas y hardening
kubectl label namespace core pod-security.kubernetes.io/enforce=restricted
```

Esta segmentacion evita tener que relajar la seguridad globalmente para acomodar un Pod que necesita privilegios especiales. Cada equipo sabe que nivel de seguridad se espera en cada namespace y configura sus Pods en consecuencia.

