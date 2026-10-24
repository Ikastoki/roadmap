# Security Context

Un contenedor ejecutandose como root dentro de un cluster es una de las superficies de ataque mas peligrosas en Kubernetes. Si un atacante compromete un contenedor con privilegios de root, puede escalar al nodo subyacente y desde alli comprometer todo el cluster. SecurityContext es el mecanismo que permite definir restricciones de seguridad a nivel de Pod y de contenedor, controlando que usuario ejecuta el proceso, que capacidades del kernel tiene disponibles, y si puede escribir en el sistema de archivos.

## Usuario no-root: `runAsNonRoot` y `runAsUser`

La medida mas basica y efectiva es impedir que el contenedor se ejecute como root. La directiva `runAsNonRoot: true` fuerza al kubelet a validar que la imagen del contenedor no ejecute su proceso principal con UID 0. Si la imagen no especifica un usuario no-root en el `Dockerfile` mediante la instruccion `USER`, el Pod no se iniciara.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-seguro
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    fsGroup: 2000
  containers:
    - name: app
      image: mi-app:1.0
```

`runAsUser` especifica el UID con el que se ejecutaran todos los procesos del contenedor, anulando cualquier usuario definido en la imagen. `fsGroup` establece el GID que se aplicara a los volumenes montados, permitiendo que el contenedor tenga los permisos adecuados sobre ellos sin necesidad de ejecutarse como root.

> Elegir un UID que no exista en el sistema host es una practica recomendada. UIDs como 1000 o 1001 suelen estar libres y evitan colisiones con usuarios del nodo.

## Capabilities de Linux: el minimo privilegio posible

Linux capabilities dividen los privilegios tradicionales de root en unidades atomicas independientes. En lugar de ejecutar un contenedor con `--privileged` (que otorga todos los capabilities), Kubernetes permite especificar exactamente que capabilities se anaden (`add`) y cuales se eliminan (`drop`) del conjunto predeterminado.

El runtime de contenedores asigna un conjunto de capabilities por defecto. La mayoria son innecesarias para aplicaciones de proposito general. La practica recomendada es eliminar todos los capabilities y anadir solo aquellos estrictamente necesarios:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

`NET_BIND_SERVICE` permite al contenedor enlazarse a puertos por debajo de 1024 sin necesidad de ejecutarse como root. Otros capabilities comunes pero peligrosos como `SYS_ADMIN` (montar sistemas de archivos, acceder a dispositivos), `SYS_PTRACE` (depurar procesos) o `SYS_RAWIO` (acceso directo a dispositivos de bloque) deben evitarse a menos que exista una justificacion solida.

```yaml
# Peligroso: otorga demasiados privilegios
securityContext:
  capabilities:
    add:
      - SYS_ADMIN
      - SYS_PTRACE
      - NET_ADMIN
      - SYS_RAWIO
```

## readOnlyRootFilesystem: sistema de archivos inmutable

Un contenedor que puede escribir en cualquier parte de su sistema de archivos es vulnerable a ataques que modifican binarios, escriben scripts maliciosos en `/tmp` o alteran archivos de configuracion. `readOnlyRootFilesystem: true` monta la capa superior del sistema de archivos del contenedor como solo lectura, impidiendo cualquier escritura fuera de los volumenes montados explicitamente.

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

Esta opcion obliga a la aplicacion a escribir solo en los directorios montados como volumenes. Es una practica excelente porque fuerza a que la aplicacion declare explicitamente donde necesita almacenar datos temporales o persistentes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-readonly
spec:
  containers:
    - name: app
      image: mi-app:1.0
      securityContext:
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

> Muchas aplicaciones asumen que `/tmp` es escribible. Montar un `emptyDir` en esa ruta es la forma mas comun de compatibilizar `readOnlyRootFilesystem` con aplicaciones existentes sin modificar su codigo.

## SecurityContext a nivel de Pod vs contenedor

El `securityContext` puede definirse a nivel de Pod y a nivel de contenedor. Las opciones definidas a nivel de Pod afectan a todos los contenedores del Pod, mientras que las definidas a nivel de contenedor solo afectan a ese contenedor y sobrescriben las del Pod en caso de conflicto.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:                      # Nivel de Pod
    runAsNonRoot: true
    fsGroup: 2000
  containers:
    - name: contenedor-a
      image: alpine
      securityContext:                  # Nivel de contenedor
        runAsUser: 1001
        capabilities:
          drop:
            - ALL
    - name: contenedor-b
      image: alpine
      securityContext:
        runAsUser: 1002
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
```

Las opciones que solo tienen sentido a nivel de contenedor son `capabilities` y `readOnlyRootFilesystem`. Opciones como `runAsUser`, `runAsGroup` y `runAsNonRoot` pueden definirse en ambos niveles, pero el valor del contenedor tiene prioridad.

## Privileged: contenedores con acceso total al host

El modo `privileged: true` otorga al contenedor todos los capabilities del kernel y acceso directo a los dispositivos del nodo, equiparandolo practicamente a un proceso ejecutandose como root en el host. Su uso debe ser excepcional y justificado: solo para componentes de infraestructura que necesiten acceso al kernel del host, como controladores CNI de red, agentes de monitorizacion que inspeccionan el sistema de archivos del nodo, o herramientas de backup.

```yaml
containers:
  - name: privileged-container
    image: my-agent
    securityContext:
      privileged: true
```

Activar `privileged` en una aplicacion de negocio es casi siempre una mala practica. Cualquier funcionalidad que parezca requerirlo puede conseguirse normalmente con capabilities especificos o montando dispositivos concretos con `volumeMounts`.

