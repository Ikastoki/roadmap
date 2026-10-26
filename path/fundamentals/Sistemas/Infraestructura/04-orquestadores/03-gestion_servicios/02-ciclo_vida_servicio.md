# Ciclo de vida de un servicio

Un servicio en un orquestador de contenedores no es una entidad estática: atraviesa una serie de fases bien definidas desde que se declara su existencia hasta que se elimina. Comprender este ciclo es esencial para diagnosticar problemas, diseñar despliegues seguros y establecer políticas de recuperación ante fallos. Cada fase del ciclo está gobernada por componentes específicos del orquestador que toman decisiones basadas en el estado actual, las condiciones del clúster y la definición declarativa del recurso.

La fase inicial es la **creación**: el usuario o un sistema automatizado envía la definición del recurso a la API del orquestador. El servidor API valida la estructura, almacena el objeto en el almacén de estado (etcd en Kubernetes) y devuelve una confirmación. A partir de ese momento, el ciclo comenzará. La siguiente fase es el **scheduleo** (programación), donde el planificador (scheduler) evalúa qué nodo del clúster tiene recursos suficientes para alojar el Pod, considerando restricciones de afinidad, tolerancias, volúmenes disponibles y políticas de antiafinidad. Una vez asignado un nodo, comienza la **ejecución**: el kubelet del nodo solicitante recibe la instrucción, descarga las imágenes necesarias, configura el entorno del contenedor, configura redes y volúmenes, y arranca el proceso.

Durante la ejecución, el orquestador **monitoriza** continuamente el estado del servicio mediante probes de salud (liveness, readiness, startup), reportes del agente en el nodo y métricas del sistema. Si el servicio necesita crecer o reducirse, entra en la fase de **escalado**, ya sea por decisión manual o automática (HPA). Cuando se actualiza la definición del servicio con una nueva imagen o configuración, comienza la fase de **actualización**, donde la estrategia definida (RollingUpdate, Blue-Green, Canary) determina cómo se reemplazan las instancias antiguas. Finalmente, cuando el recurso se elimina, el orquestador ejecuta la fase de **destrucción**, deteniendo contenedores, liberando recursos y limpiando objetos asociados.

Los Pods en Kubernetes atraviesan una serie de estados que reflejan su posición en este ciclo:

| Estado | Descripcion |
|--------|-------------|
| `Pending` | Aceptado por el API pero aun no programado o en proceso de descarga de imagenes |
| `Running` | Programado en un nodo, todos los contenedores estan en ejecucion al menos uno sigue activo |
| `Succeeded` | Todos los contenedores terminaron con exito (exit code 0) y no se reiniciaran |
| `Failed` | Al menos un contenedor termino con un codigo de fallo o fue terminado por el sistema |
| `Unknown` | El estado no pudo determinarse, tipicamente por fallo de comunicacion con el nodo |

```bash
# Ver el estado de todos los Pods en un namespace
kubectl get pods -n produccion

# Salida ejemplo:
# NAME                        READY   STATUS    RESTARTS   AGE
# api-gateway-7d4f8b6c-x2k9p  1/1     Running   0          3d
# api-gateway-7d4f8b6c-m8j2q  1/1     Running   0          3d
# api-gateway-7d4f8b6c-n5l3r  1/1     Running   0          3d
# worker-queue-a1b2c           0/1     Pending   0          2m

# Obtener informacion detallada de un Pod especifico
kubectl describe pod api-gateway-7d4f8b6c-x2k9p -n produccion

# Ver el estado de los Deployments y su disponibilidad
kubectl get deploy -n produccion
# NAME           READY   UP-TO-DATE   AVAILABLE   AGE
# api-gateway    3/3     3            3           5d

# Ver eventos recientes del clúster para diagnosticar problemas
kubectl get events -n produccion --sort-by='.lastTimestamp' | tail -20
```

El estado `Unknown` merece atención especial. Ocurre cuando el nodo que aloja un Pod deja de enviar heartbeats al servidor API. El controlador NodeLifeCycle espera un período configurable (por defecto, 5 minutos llamado `pod-eviction-timeout`) antes de marcar el nodo como `NotReady` y beginar la evacuación de sus Pods. Si el nodo se recupera antes de ese período, los Pods vuelven a estado `Running` sin intervención manual.

> El estado `Succeeded` es frecuente en tareas tipo Job (migraciones de base de datos, procesamiento por lotes). Un Pod en estado `Succeeded` no se considera un fallo, incluso si el contenedor terminó. Esto es relevante al diseñar alertas: no se debe alertar por Pods que completan su trabajo esperado.

> El campo `RESTARTS` en la salida de `kubectl get pods` es un indicador valioso. Un Pod con múltiples reinicios en un período corto sugiere un problema de liveness probe (el contenedor se reinicia porque falla el chequeo de salud) o un fallo interno (panic, OOMKilled). Inspeccionar los eventos con `kubectl describe` o los logs con `kubectl logs --previous` revela la causa.

