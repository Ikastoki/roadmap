# Gestion de configuracion en orquestadores

La separacion entre codigo fuente y configuracion es uno de los principios fundamentales del desarrollo de software moderno, y en entornos containerizados esta separacion adquiere una relevancia critica. Un mismo contenedor de aplicacion debe poder desplegarse en distintos entornos (desarrollo, pruebas, produccion) sin necesidad de reconstruir la imagen, simplemente variando los valores de configuracion que inyecta el orquestador. En Kubernetes, los ConfigMaps (o Configs) proporcionan un mecanismo nativo para almacenar datos de configuracion no sensibles de forma declarativa, permitiendo que los pods consuman estas configuraciones de manera transparente, ya sea como variables de entorno o como ficheros montados en el sistema de archivos del contenedor. Esta abstraccion elimina la necesidad de incrustar valores de configuracion en las imagenes Docker, reduciendo la superficie de ataque y facilitando la gestion centralizada de configuraciones a traves de multiples despliegues.

## Estructura de un ConfigMap

Un ConfigMap es un recurso de tipo API que almacena pares clave-valor donde los valores pueden ser cadenas de texto, ficheros de configuracion completos o contenido JSON/YAML. El orquestador almacena el ConfigMap en etcd junto con el resto de recursos del cluster, lo que permite su recuperacion rapida y su replicacion entre nodos. Los ConfigMaps no deben contener datos sensibles como contrasenas o tokens; para esos casos existe el recurso Secret, que veremos posteriormente. La definicion de un ConfigMap especifica explicitamente los datos que contiene, y estos datos pueden provenir de multiples fuentes: literales definidos en la manifestacion, ficheros existentes en el sistema de archivos, o directorios completos.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: produccion
  labels:
    app: mi-aplicacion
    entorno: produccion
data:
  # Valores literales como pares clave-valor
  DATABASE_HOST: "postgresql.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "app_production"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  
  # Fichero de configuracion completo
  application.yml: |
    server:
      port: 8080
      servlet:
        context-path: /api
    spring:
      datasource:
        url: jdbc:postgresql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
      redis:
        host: redis.cluster.local
        port: 6379
    logging:
      level:
        root: ${LOG_LEVEL}
        org.springframework.web: DEBUG

  # Configuracion nginx como fichero
  nginx.conf: |
    upstream backend {
        server 127.0.0.1:8080;
    }
    
    server {
        listen 80;
        server_name api.example.com;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
```

> Los ConfigMaps con datos grandes (mas de 256KB) no son recomendables. Para configuraciones extensas, considera montar ficheros desde almacenamiento externo o dividir la configuracion en multiples ConfigMaps tematicos.

## Inyeccion de configuracion en Pods

El orquestador ofrece dos mecanismos principales para consumir un ConfigMap dentro de un pod: como variables de entorno o como ficheros montados en un punto especifico del sistema de archivos. Cada mecanismo tiene implicaciones distintas sobre la disponibilidad inmediata de los valores, la capacidad de actualizacion sin reinicio y la compatibilidad con diferentes tipos de aplicaciones.

### Como variables de entorno

Cuando se inyecta un ConfigMap como variable de entorno, el valor se captura en el momento de la creacion del pod y se inyecta en el entorno del contenedor. Este enfoque es compatible con praticamente cualquier aplicacion, ya que la mayoria de lenguajes de programacion leen variables de entorno mediante mecanismos estandar. Sin embargo, una limitacion importante es que las variables de entorno no se actualizan dinamicamente si el ConfigMap cambia despues de la creacion del pod; el contenedor debe ser recreado para recibir los nuevos valores.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: produccion
spec:
  containers:
    - name: aplicacion
      image: mi-app:2.1.0
      envFrom:
        # Inyectar todas las claves del ConfigMap como variables de entorno
        - configMapRef:
            name: app-config
      # O seleccionar claves especificas
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_PORT
              optional: true  # No falla si la clave no existe
```

### Como ficheros montados

El segundo mecanismo consiste en montar el ConfigMap como un volumen en un directorio del contenedor. Cada clave del ConfigMap se convierte en un fichero dentro de ese directorio, y el contenido del valor es el contenido del fichero. Esta aproximacion es ideal para ficheros de configuracion como `nginx.conf`, `application.yml` o cualquier otro formato que la aplicacion espere leer del disco. Ademas, el orquestador actualiza automaticamente los ficheros montados cuando el ConfigMap cambia (con un retardo configurable), lo que permite a algunas aplicaciones recargar su configuracion sin necesidad de reiniciar el contenedor.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: produccion
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
        # Opcional: filtrar claves especificas
        items:
          - key: nginx.conf
            path: default.conf  # Nombre del fichero en el punto de montaje
```

> Cuando el ConfigMap se monta como ficheros, el orquestador usa symlinks para apuntar a la version actual del dato. La actualizacion de los ficheros montados ocurre con un retardo de hasta un minuto por defecto, configurable mediante el campo `kubeletConfig` en la configuracion del nodo.

## Actualizacion dinamica de configuraciones

Una de las ventajas mas significativas de los ConfigMaps montados como volumenes es la capacidad de actualizar la configuracion sin reiniciar el contenedor. Cuando se modifica un ConfigMap que esta siendo consumido por pods via volumenes, el kubelet detecta el cambio y actualiza los ficheros montados en los contenedores correspondientes. Este mecanismo utiliza inotify a nivel del kernel para notificar a las aplicaciones que el fichero ha cambiado, permitiendoles recargar su configuracion si estan disenadas para ello.

```bash
# Crear un ConfigMap desde un directorio local
kubectl create configmap app-config \
  --from-file=./config/ \
  --namespace=produccion

# Actualizar el ConfigMap (el orquestador propaga los cambios)
kubectl create configmap app-config \
  --from-file=./config/nueva_config.yml \
  --namespace=produccion \
  --dry-run=client -o yaml | kubectl apply -f -

# Verificar que los ficheros montados se actualizaron
kubectl exec -it nginx-pod -- ls -la /etc/nginx/conf.d/
kubectl exec -it nginx-pod -- cat /etc/nginx/conf.d/default.conf
```

> No todos los binarios responden a la senal SIGHUP o a eventos inotify. Muchas aplicaciones necesitan un mecanismo explicito de recarga de configuracion, como una endpoint `/actuator/refresh` en Spring Boot o una signal personalizada manejada por el proceso principal.

## Buena practica: gestor de configuracion externo

Para entornos de produccion con multiples servicios, es recomendable organizar los ConfigMaps por dominio funcional y mantener un patron de nomenclatura consistente. La herramienta `kustomize`, integrada en `kubectl`, permite generar ConfigMaps a partir de ficheros locales y gestionar variantes de configuracion por entorno sin duplicar manifestaciones.

```yaml
# kustomization.yaml - gestor de configuraciones por entorno
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
  - name: app-config
    files:
      - configs/application.yml
      - configs/nginx.conf
    options:
      # Anadir sufijo con hash para forzar reinicio de pods
      disableNameSuffixHash: false

  - name: app-secrets-config
    literals:
      - DATABASE_HOST=postgresql.cluster.local
      - REDIS_HOST=redis.cluster.local

# Aplicar configuracion por entorno
# kubectl apply -k overlays/produccion/
```

