# ConfigMaps

Una aplicacion rara vez funciona con la misma configuracion en todos los entornos. La base de datos de desarrollo no es la de produccion, los niveles de log cambian, las URLs de servicios externos varian. Antes de Kubernetes, lo habitual era construir la configuracion dentro de la imagen del contenedor o pasarla mediante variables de entorno en el momento de arrancar el contenedor. El primer enfoque rompe la separacion entre codigo y configuracion; el segundo funciona pero no escala cuando decenas de contenedores comparten los mismos valores.

ConfigMap es el recurso de Kubernetes que almacena configuracion no sensible en forma de pares clave-valor o como archivos de configuracion completos. Su proposito es desacoplar los artefactos de configuracion de la imagen del contenedor, permitiendo que un mismo contenedor se ejecute en distintos entornos simplemente cambiando el ConfigMap al que hace referencia.

## Creacion de ConfigMaps

Un ConfigMap se puede crear de forma imperativa con kubectl o de forma declarativa mediante un manifiesto YAML. La via imperativa es util para desarrollo y pruebas rapidas; la declarativa es la recomendada para entornos gestionados bajo control de versiones.

```bash
# Desde literales clave=valor
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=DB_HOST=postgres.prod.svc.cluster.local

# Desde archivos (cada archivo se convierte en una clave con su contenido)
kubectl create configmap nginx-config --from-file=nginx.conf

# Desde un directorio completo (cada archivo del directorio es una clave)
kubectl create configmap app-config --from-file=config/

# Combinando fuentes
kubectl create configmap mi-config \
  --from-literal=APP_ENV=production \
  --from-file=app.properties
```

La forma declarativa ofrece mas control y es reproducible:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: debug
  DB_HOST: postgres.prod.svc.cluster.local
  DB_PORT: "5432"
  app.properties: |
    cache.ttl=300
    cache.max-size=1024
    retry.max-attempts=3
```

El campo `data` contiene las claves y valores en texto plano. Kubernetes no impone restricciones sobre el contenido: puede ser un valor corto, un archivo de propiedades, un JSON o incluso un script. La unica limitacion es que el valor debe ser una cadena UTF-8 valida.

> Los nombres de las claves en un ConfigMap deben ser validos como nombres de variables de entorno si se van a inyectar como tales. Esto significa que no pueden empezar por un numero y los guiones no son compatibles con todas las shells. La convencion es usar MAYUSCULAS_CON_GUIONES_BAJOS.

## Inyeccion como variables de entorno

Una vez creado el ConfigMap, se inyecta en un Pod referenciandolo en el campo `envFrom` o `env` del contenedor. Con `envFrom` se importan todas las claves del ConfigMap como variables de entorno de una sola vez:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: mi-app:1.0
      envFrom:
        - configMapRef:
            name: app-config
```

Si solo se necesita un subconjunto de las claves, o si se quiere asignar un nombre distinto a la variable de entorno, se usa `env` con `valueFrom`:

```yaml
spec:
  containers:
    - name: app
      image: mi-app:1.0
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

Cuando se inyecta un ConfigMap como variable de entorno, los cambios posteriores en el ConfigMap no se reflejan en los Pods ya en ejecucion. Las variables de entorno se fijan en el momento de arrancar el contenedor. Para que un Pod recoja cambios en la configuracion, debe ser recreado.

## Inyeccion como volumenes

Montar un ConfigMap como volumen permite que el contenido se actualice dinamicamente mientras el Pod esta en ejecucion. Cada clave del ConfigMap se convierte en un archivo dentro del punto de montaje, con el nombre de la clave como nombre del archivo y su valor como contenido:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: mi-app:1.0
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: app-config
```

Con el manifiesto anterior, el Pod tendria los archivos `/etc/config/LOG_LEVEL`, `/etc/config/DB_HOST`, `/etc/config/DB_PORT` y `/etc/config/app.properties`, cada uno con el valor correspondiente.

Kubernetes sincroniza periodicamente el contenido del volumen con el ConfigMap. La sincronizacion no es instantanea pero suele ocurrir en menos de un minuto. Esto permite cambiar la configuracion sin reiniciar el Pod, aunque la aplicacion debe estar disenada para detectar cambios en los archivos de configuracion (por ejemplo, mediante un watcher de sistema de archivos).

```yaml
# Montar solo claves especificas con rutas personalizadas
volumes:
  - name: config
    configMap:
      name: app-config
      items:
        - key: app.properties
          path: config.properties
```

> Montar un ConfigMap como volumen no elimina archivos preexistentes en el directorio de montaje a menos que se use `mountPath` con un directorio vacio. Si el directorio ya contiene archivos (por ejemplo, `/etc/nginx`), estos quedaran ocultos. La solucion es montar en un subdirectorio o usar `subPath`.

## Proyeccion de ConfigMaps

Kubernetes permite combinar multiples fuentes de configuracion en un unico volumen mediante el recurso `ProjectedVolume`. Es util cuando un Pod necesita consumir datos de varios ConfigMaps y Secrets simultaneamente:

```yaml
volumes:
  - name: combined-config
    projected:
      sources:
        - configMap:
            name: app-config
        - configMap:
            name: logging-config
        - secret:
            name: db-credentials
```

Cada fuente se proyecta en el mismo volumen sin colisiones de nombres, siempre que las claves no se solapen.

## Buenas practicas

Los ConfigMaps deben nombrarse de forma descriptiva e incluir el entorno o el proposito en el nombre (por ejemplo, `api-config-prod`, `frontend-config-staging`). No incluir informacion sensible en un ConfigMap es la regla fundamental: cualquier valor que sea una contrasena, un token, una clave API o un certificado pertenece a un Secret, no a un ConfigMap.

Es recomendable versionar los ConfigMaps junto con el codigo en el repositorio. La estrategia tipica es tener un directorio `k8s/configmaps/` con los manifiestos YAML para cada entorno. En flujos GitOps (ArgoCD, Flux), los ConfigMaps se sincronizan automaticamente desde el repositorio al cluster.

Cuando se usa el patron de ConfigMap montado como volumen para configuracion dinamica, la aplicacion debe implementar recarga en caliente. No todas las aplicaciones soportan este comportamiento; para las que no, la opcion es recrear los Pods cuando la configuracion cambia. Herramientas como Reloader (stakater) pueden detectar cambios en ConfigMaps y reiniciar automaticamente los Deployments asociados.

```bash
# Ver ConfigMaps
kubectl get configmaps

# Ver detalles de un ConfigMap
kubectl describe configmap app-config

# Ver los datos en formato YAML
kubectl get configmap app-config -o yaml
```

