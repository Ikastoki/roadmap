# Comandos esenciales de kubectl

kubectl sigue un patron consistente en todos sus comandos: `kubectl <verbo> <recurso> [nombre] [flags]`. Este esquema, inspirado en la arquitectura REST del API server, permite que una vez comprendida la estructura basica, cualquier recurso de Kubernetes se maneje con la misma sintaxis. Verbo puede ser `get`, `describe`, `delete`, `apply`, `logs` o `exec`; recurso puede ser `pods`, `nodes`, `services`, `deployments`, y cualquier recurso personalizado (CRD) que extienda el API.

La mayoria de los comandos aceptan el flag `-o` (output) para controlar el formato de salida: `wide` (columnas extendidas), `json` (estructura completa), `yaml` (version YAML del recurso), o `name` (solo nombres). Esta flexibilidad hace que kubectl sea util tanto para inspeccion visual como para scripting.

## kubectl get

`kubectl get` es el comando mas utilizado. Lista recursos del cluster con informacion resumida en formato tabular. Sin argumentos adicionales, opera en el namespace del context activo (normalmente `default`).

```bash
# Ver todos los nodos del cluster
kubectl get nodes

# Ver todos los pods en el namespace actual
kubectl get pods

# Ver pods con detalles adicionales (nodo, IP)
kubectl get pods -o wide

# Ver todos los recursos de un tipo en todos los namespaces
kubectl get pods --all-namespaces
# o su alias
kubectl get pods -A
```

La columna `STATUS` de los pods indica su estado actual. Los valores mas comunes son:

- `Running`: el pod esta ejecutandose correctamente
- `Pending`: el pod ha sido aceptado pero aun no se ha iniciado (descargando imagen, esperando recursos)
- `CrashLoopBackOff`: el contenedor se inicia, falla y Kubernetes lo reintenta continuamente
- `Error`: el contenedor termino con codigo de error
- `ImagePullBackOff`: no se pudo descargar la imagen del contenedor

```bash
# Ver pods con su nodo asignado y IP
kubectl get pods -o wide
# NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE
# nginx-7c79c4bf97-8kz2h   1/1     Running   0          10m   10.244.1.5    worker-1

# Ver recursos por etiqueta
kubectl get pods -l app=nginx

# Ver solo los nombres (util para scripting)
kubectl get pods -o name
```

El flag `-o wide` es el primer nivel de detalle. Anyade columnas como la IP del pod y el nodo donde se ejecuta, informacion que no aparece en la salida por defecto pero que es esencial para depurar problemas de red o de asignacion de recursos.

Para obtener la representacion completa de un recurso en formato JSON o YAML:

```bash
# Salida YAML completa
kubectl get pod nginx-7c79c4bf97-8kz2h -o yaml

# Salida JSON
kubectl get pod nginx-7c79c4bf97-8kz2h -o json

# Filtrar campos especificos con jsonpath
kubectl get pod nginx-7c79c4bf97-8kz2h -o jsonpath='{.status.podIP}'
```

> `-o json` combinado con herramientas como `jq` permite consultas complejas sobre el estado del cluster. Por ejemplo, `kubectl get pods -o json | jq '.items[] | {name: .metadata.name, ip: .status.podIP}'` lista todos los pods con sus IPs en formato estructurado.

## kubectl describe

Mientras que `get` muestra un resumen tabular, `describe` proporciona informacion detallada de un recurso: eventos recientes, montajes de volumen, condiciones, puertos expuestos, etiquetas, anotaciones, y el historial de cambios de estado.

```bash
# Descripcion detallada de un nodo
kubectl describe node worker-1

# Descripcion detallada de un pod
kubectl describe pod nginx-7c79c4bf97-8kz2h

# Descripcion de un deployment
kubectl describe deployment nginx
```

La seccion mas valiosa de `describe` son los _Events_ al final de la salida. Esta seccion registra todos los eventos relacionados con el recurso: programacion de pods, descarga de imagenes, errores de montaje, reinicios. Cuando un pod no arranca, los eventos suelen contener la causa raiz.

```bash
# Ejemplo de salida de describe pod
# Events:
#   Type     Reason     Age   From               Message
#   ----     ------     ----  ----               -------
#   Normal   Scheduled  5m    default-scheduler  Successfully assigned default/nginx to worker-1
#   Normal   Pulling    5m    kubelet            Pulling image "nginx:latest"
#   Normal   Pulled     4m    kubelet            Successfully pulled image
#   Normal   Created    4m    kubelet            Created container nginx
#   Normal   Started    4m    kubelet            Started container nginx
```

> Cuando un recurso no se comporta como esperas, ejecuta `kubectl describe` primero. Los eventos muestran la secuencia completa de acciones y errores, algo que `kubectl get` no proporciona. Si el pod esta en `Pending`, describe suele revelar si falta CPU, memoria o si la imagen no existe.

## kubectl logs

`kubectl logs` recupera la salida estandar (stdout/stderr) de un contenedor en ejecucion. Es la herramienta principal de debugging para aplicaciones dentro de Kubernetes, equivalente a `docker logs` pero adaptada al ecosistema de pods.

```bash
# Ver logs de un pod (un solo contenedor)
kubectl logs nginx-7c79c4bf97-8kz2h

# Seguir logs en tiempo real (streaming)
kubectl logs -f nginx-7c79c4bf97-8kz2h

# Ultimas N lineas
kubectl logs --tail=50 nginx-7c79c4bf97-8kz2h

# Logs con timestamp
kubectl logs --timestamps nginx-7c79c4bf97-8kz2h
```

Si el pod contiene multiples contenedores, es obligatorio especificar cual con el flag `-c`:

```bash
# Pod con multiples contenedores
kubectl logs nginx-sidecar-7c79c4bf97-8kz2h -c sidecar-container

# Seguir logs de todos los contenedores de un pod
kubectl logs -f nginx-sidecar-7c79c4bf97-8kz2h --all-containers
```

Para pods que ya no existen (terminados o reiniciados), el flag `-p` (previous) recupera los logs del contenedor anterior:

```bash
# Logs del contenedor anterior (antes del reinicio)
kubectl logs -p nginx-7c79c4bf97-8kz2h
```

> Los logs de `kubectl logs` solo capturan stdout/stderr del contenedor. Si la aplicacion escribe logs en archivos dentro del contenedor, esos no son accesibles mediante este comando. Para ese caso es necesario usar `kubectl exec` o configurar un sidecar que redirija los archivos a stdout.

## kubectl exec

`kubectl exec` ejecuta un comando dentro de un contenedor en ejecucion. Es la puerta de entrada para inspeccionar el sistema de archivos, ejecutar herramientas de diagnostico o modificar configuraciones temporales.

```bash
# Ejecutar un comando y salir
kubectl exec nginx-7c79c4bf97-8kz2h -- ls /var/log

# Terminal interactiva
kubectl exec -it nginx-7c79c4bf97-8kz2h -- /bin/sh

# Ejecutar en un contenedor especifico (multi-contenedor)
kubectl exec -it nginx-7c79c4bf97-8kz2h -c sidecar-container -- /bin/sh
```

El doble guion `--` separa los argumentos de kubectl del comando a ejecutar dentro del contenedor. Todo lo que va despues de `--` se pasa directamente al contenedor sin ser interpretado por kubectl.

```bash
# Ejecutar un comando con argumentos
kubectl exec nginx-7c79c4bf97-8kz2h -- ls -la /app/config

# Variables de entorno dentro del contenedor
kubectl exec nginx-7c79c4bf97-8kz2h -- env

# Ver procesos en ejecucion
kubectl exec nginx-7c79c4bf97-8kz2h -- ps aux
```

El flag `-it` combina `-i` (stdin abierto, permite enviar entrada) y `-t` (terminal virtual, permite mostrar correctamente programas interactivos). Sin `-t`, comandos como `sh` o `top` no renderizan correctamente la interfaz.

> `kubectl exec` requiere que el contenedor tenga `/bin/sh` o `/bin/bash` disponibles. Las imagenes minimizadas (alpine, distroless) pueden no incluir un shell. En esos casos el comando debe ser el binario directamente: `kubectl exec pod -- /my-binary --version`.

## Flags comunes y combinaciones

```bash
# Ver recursos por namespace
kubectl get pods -n kube-system

# Ver frecuencia de actualizacion (watch)
kubectl get pods -w

# Combinar -w con -o wide
kubectl get pods -o wide -w

# Usar contexto especifico sin cambiar el context activo
kubectl --context=production get pods
```

El flag `-w` (watch) mantiene la conexion abierta y actualiza la salida en tiempo real ante cualquier cambio. Es util durante despliegues o cuando se espera que un recurso cambie de estado.

