# Glosario de conceptos clave

El ecosistema de contenedores tiene su propio vocabulario que puede resultar abrumador al principio. Este glosario reune los conceptos mas importantes con definiciones claras para tener una referencia rapida.

## Conceptos fundamentales

- **Contenedor:** Instancia en ejecucion de una imagen. Es un proceso aislado del sistema host.
- **Imagen:** Plantilla inmutable de solo lectura que contiene el SO, las dependencias y el codigo de la aplicacion. Es el molde a partir del cual se crean contenedores.
- **Container Runtime:** Software de bajo nivel que ejecuta el contenedor directamente (ej. runc, crun, crun).
- **Container Engine:** Software que gestiona el ciclo de vida de imagenes y contenedores, se comunica con el runtime y proporciona la API (ej. Docker Engine, Podman, containerd).
- **Registry:** Servicio que almacena y distribuye imagenes (ej. Docker Hub, Harbor, AWS ECR).

## Archivos y configuracion

- **Dockerfile / Containerfile:** Archivo de texto con instrucciones para construir una imagen. Docker usa `Dockerfile`; Podman usa `Containerfile` (son compatibles).
- **Contexto de construccion:** Directorio enviado al motor de contenedores que contiene el Dockerfile y los archivos que necesita la imagen.
- **`.dockerignore`:** Archivo que excluye archivos del contexto de construccion (similar a `.gitignore`).

## Capas y construccion

- **Layer (capa):** Cada instruccion del Dockerfile genera una capa de solo lectura. Las capas se apilan para formar la imagen final.
- **Build:** Proceso de construccion de una imagen desde un Dockerfile.
- **Cache de capas:** Las capas no modificadas se reutilizan entre construcciones, acelerando el proceso.
- **Multi-stage build:** Dockerfile con multiples instrucciones `FROM` para compilar en una etapa y copiar solo el artefacto final a una imagen minima de produccion.

## Ejecucion y gestion

- **Tag (etiqueta):** Nombre legible que apunta a un digest de imagen especifico (ej. `nginx:1.25-alpine`).
- **Digest:** Identificador unico y criptografico de una imagen (sha256:...).
- **Entrypoint:** Comando que se ejecuta siempre al iniciar el contenedor. No se puede sobreescribir con argumentos.
- **CMD:** Comando por defecto que se ejecuta al iniciar el contenedor. Se puede sobreescribir con argumentos.
- **`-d` (detached):** Ejecutar el contenedor en segundo plano (sin adjuntar la terminal).
- **`-p` (publish):** Mapear puertos del host a puertos del contenedor.
- **`-e` (environment):** Definir variables de entorno dentro del contenedor.

## Almacenamiento

- **Volumen (Volume):** Almacenamiento gestionado por el motor de contenedores. Persiste despues de eliminar el contenedor. Es la opcion recomendada para produccion.
- **Bind Mount:** Vincular un directorio del host al contenedor. Ideal para desarrollo (hot reload).
- **tmpfs:** Almacenamiento temporal en memoria RAM. Solo en Linux.

## Redes

- **Bridge:** Red aislada predeterminada. Los contenedores se comunican entre si y con Internet a traves de NAT.
- **Host:** El contenedor comparte la red del host sin aislamiento.
- **None:** Sin interfaz de red.
- **Overlay:** Red que permite comunicacion entre contenedores en diferentes hosts (clusters).
- **Macvlan:** Asigna una MAC real al contenedor, apareciendo como un dispositivo fisico en la red.

## Orquestacion

- **Orquestacion:** Gestion automatica de multiples contenedores: escalado, despliegue, red, almacenamiento, monitoreo.
- **Kubernetes (K8s):** El orquestador de contenedores mas popular. Gestiona clusters de hosts que ejecutan contenedores.
- **Pod:** Unidad minima de despliegue en Kubernetes. Contiene uno o mas contenedores que comparten red y almacenamiento.
- **Deployment:** Objeto de Kubernetes que define el estado deseado de los contenedores (cuantos replicas, que imagen, que recursos).
- **Service:** Abstraccion de red en Kubernetes que expone un conjunto de pods como un servicio con una IP estable.

## Seguridad

- **Rootless:** Ejecutar contenedores sin privilegios de root. Mejora significativamente la seguridad.
- **Non-root:** El proceso dentro del contenedor se ejecuta con un usuario no privilegiado.
- **Seccomp, AppArmor, SELinux:** Perfiles de seguridad que restringen las llamadas al sistema que un contenedor puede realizar.

> Este glosario es una referencia rapida. Los conceptos se explican con mas detalle en las secciones correspondientes. Lo importante es entender que el ecosistema de contenedores tiene un vocabulario coherente y bien definido que facilita la comunicacion entre profesionales.

