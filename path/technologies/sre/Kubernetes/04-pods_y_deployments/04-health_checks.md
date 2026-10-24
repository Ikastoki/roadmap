# Health Checks

Un contenedor puede estar ejecutandose pero ser incapaz de servir trafico. Un proceso colgado en un bucle infinito, una conexion a base de datos perdida, o un deadlock interno son ejemplos de fallos que no provocan la terminacion del proceso pero hacen que la aplicacion no responda correctamente. Kubernetes no puede detectar estos estados sin ayuda explicita del desarrollador.

Las health probes son los mecanismos que Kubernetes proporciona para que el kubelet evalue la salud de los contenedores en ejecucion. Se definen en la especificacion del contenedor y se ejecutan periodicamente mientras el contenedor esta activo. Segun el resultado de la prueba, Kubernetes decide si reiniciar el contenedor, si incluirlo en el balanceo de carga, o si esperar antes de considerarlo listo.

## Liveness probe

La liveness probe determina si un contenedor esta vivo. Si falla, el kubelet mata el contenedor y lo reinicia segun la politica de reinicio del pod (por defecto `Always`). Su proposito es recuperar contenedores que entraron en un estado irrecuperable pero cuyo proceso sigue ejecutandose.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: app
      image: mi-app
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 3
```

`initialDelaySeconds` retrasa la primera ejecucion de la probe para dar tiempo al contenedor a iniciarse. `periodSeconds` define el intervalo entre ejecuciones. `timeoutSeconds` es el tiempo maximo que la probe espera una respuesta. `failureThreshold` es el numero de fallos consecutivos necesarios para considerar que la probe ha fallado.

La liveness probe debe ser ligera. Una probe que hace consultas pesadas a la base de datos o que recorre estructuras grandes puede degradar el rendimiento del contenedor y del propio kubelet. La recomendacion general es que la liveness probe solo verifique que el proceso principal responde, sin dependencias externas.

> Una liveness probe demasiado agresiva puede causar un bucle de reinicios (crash loop backoff) si la aplicacion tarda en arrancar o si tiene picos de latencia. Ajusta `initialDelaySeconds` y `failureThreshold` con margen suficiente para el tiempo de arranque real de la aplicacion.

## Readiness probe

La readiness probe determina si un contenedor esta listo para recibir trafico. Si falla, el contenedor se elimina del conjunto de endpoints del Service, pero no se reinicia. Cuando la probe vuelve a tener exito, el contenedor se reincorpora al balanceo de carga.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
spec:
  containers:
    - name: app
      image: mi-app
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 5
        timeoutSeconds: 2
        successThreshold: 2
```

`successThreshold` define cuantos exitos consecutivos se requieren para considerar la probe como exitosa despues de un fallo. Por defecto es 1. Para aplicaciones que necesitan estabilizarse antes de recibir trafico, un valor de 2 o 3 evita fluctuaciones en el balanceo.

La readiness probe tipicamente verifica dependencias externas: conexion a base de datos, disponibilidad de servicios upstream, carga de datos iniciales, o que el pool de conexiones este caliente. A diferencia de la liveness probe, la readiness puede ser mas pesada porque no se ejecuta con tanta frecuencia y su fallo no provoca reinicios.

La combinacion de ambas probes permite escenarios como:

- El contenedor arranca, la liveness probe pasa porque el proceso responde, pero la readiness probe falla porque la base de datos aun no esta disponible. El pod no recibe trafico hasta que la base de datos responde.
- Durante un pico de carga, el contenedor empieza a rechazar peticiones. La readiness probe falla, el pod sale del balanceo, y las peticiones se redirigen a otros pods. Cuando la carga se normaliza, la readiness probe vuelve a tener exito y el pod reingresa al servicio.

## Startup probe

La startup probe se introdujo en Kubernetes 1.18 para resolver un problema comun: aplicaciones que tardan mucho en arrancar (minutos en lugar de segundos). Sin startup probe, la liveness probe se ejecutaria durante el arranque y, si la aplicacion tarda demasiado, la mataria prematuramente.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
    - name: app
      image: mi-app
      startupProbe:
        httpGet:
          path: /startup
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 30
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
```

La startup probe se ejecuta una unica vez al inicio del contenedor. Mientras la startup probe no tenga exito, las otras probes (liveness y readiness) estan desactivadas. Cuando la startup probe finalmente tiene exito, las probes de liveness y readiness retoman el control con sus configuraciones normales.

En el ejemplo anterior, con `failureThreshold: 30` y `periodSeconds: 10`, el contenedor tiene hasta 300 segundos (5 minutos) para arrancar antes de que la startup probe falle y el kubelet lo reinicie. Una vez arrancado, la liveness probe tiene un threshold de 3 fallos (30 segundos) para detectar problemas.

## Tipos de probes

Kubernetes soporta tres tipos de handlers para las probes. Cada uno es adecuado para distintos escenarios.

### HTTP probe

Realiza una peticion HTTP GET a una ruta y puerto del contenedor. La probe tiene exito si el codigo de respuesta esta entre 200 y 399. Es el tipo mas comun para aplicaciones web y APIs REST.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: X-Health-Check
        value: "true"
  initialDelaySeconds: 5
  periodSeconds: 10
```

El campo `httpHeaders` permite anadir cabeceras personalizadas a la peticion. Es util si la aplicacion requiere autenticacion o un header especifico para el endpoint de health check.

### TCP probe

Intenta establecer una conexion TCP al puerto especificado. La probe tiene exito si el puerto acepta la conexion. Es util para aplicaciones que usan protocolos no HTTP (gRPC, bases de datos, colas de mensajes) o cuando no se puede exponer un endpoint HTTP de health check.

```yaml
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 10
  periodSeconds: 5
```

TCP probe verifica que el proceso esta escuchando en el puerto, pero no confirma que la aplicacion este funcional. Un proceso puede tener el puerto abierto pero estar en un estado degradado. Para verificaciones mas precisas, es preferible HTTP probe cuando sea posible.

### Exec probe

Ejecuta un comando dentro del contenedor. La probe tiene exito si el comando termina con codigo de salida 0, y falla con cualquier otro codigo. Es el tipo mas flexible porque permite cualquier logica de verificacion que pueda ejecutarse dentro del contenedor.

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - |
        if [ -f /tmp/healthy ]; then
          exit 0
        else
          exit 1
        fi
  initialDelaySeconds: 5
  periodSeconds: 15
```

Exec probe es util cuando la aplicacion no expone un puerto HTTP o TCP, o cuando se necesita verificar condiciones internas del contenedor (existencia de un archivo, resultado de un script, estado de un proceso hijo).

> Las exec probe ejecutan el comando dentro del contenedor, consumiendo recursos del mismo. Si el comando es pesado (por ejemplo, un script que consulta la base de datos), puede competir por CPU con el proceso principal. Manten las exec probes ligeras y con timeout ajustado.

## Politica de reinicio

El campo `restartPolicy` del pod determina como reacciona Kubernetes cuando un contenedor termina. Se aplica a todos los contenedores del pod y tiene tres valores posibles:

```yaml
spec:
  restartPolicy: Always   # por defecto
```

`Always`: el kubelet reinicia el contenedor siempre que termina, independientemente del codigo de salida. Es la politica por defecto para Deployments y la adecuada para aplicaciones que deben estar siempre ejecutandose.

`OnFailure`: reinicia el contenedor solo si termina con codigo de salida distinto de cero. Util para Jobs donde los contenedores deben ejecutarse una vez y solo reintentar si fallan.

`Never`: no reinicia el contenedor. El pod queda en estado `Completed` (exito) o `Error` (fallo). Util para tareas batch que solo deben ejecutarse una vez.

Cuando un contenedor se reinicia repetidamente, Kubernetes aplica un backoff exponencial: 10s, 20s, 40s, 80s... hasta un maximo de 300 segundos (5 minutos). Este backoff se restablece cuando el contenedor permanece sano durante 10 minutos.

```bash
# Ver los reinicios de un pod
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# api-6f8d4c8b7c-abc12    1/1     Running   3          15m

# Ver eventos de reinicio
kubectl describe pod api-6f8d4c8b7c-abc12
# Events:
#   Type     Reason     Age                  From               Message
#   ----     ------     ----                 ----               -------
#   Warning  BackOff    1m (x5 over 5m)      kubelet            Back-off restarting failed container
```

## Configuracion recomendada

No existe una configuracion universal de health probes que funcione para todas las aplicaciones. Los valores optimos dependen del tiempo de arranque, la latencia tipica, la criticidad del servicio y la tolerancia a falsos positivos.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 15
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3
```

Reglas generales:

- La readiness probe debe ser mas frecuente que la liveness probe para reaccionar rapidamente a degradaciones.
- La liveness probe debe tener un `initialDelaySeconds` mayor que el tiempo de arranque real de la aplicacion.
- Los timeouts deben ser suficientemente amplios para evitar falsos positivos por latencia, pero no tanto como para retrasar la deteccion de fallos reales.
- Si la aplicacion tarda mas de 60 segundos en arrancar, utiliza startup probe con un `failureThreshold` alto.
- No uses la liveness probe para verificar dependencias externas (bases de datos, caches). Eso es responsabilidad de la readiness probe. Si la liveness probe falla por una dependencia externa caida, se reiniciara el contenedor sin necesidad, agravando el problema.

