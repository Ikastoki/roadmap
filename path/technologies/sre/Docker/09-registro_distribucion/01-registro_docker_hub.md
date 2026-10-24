# Registro y distribucion de imagenes: Docker Hub

Una imagen Docker almacenada localmente tiene valor unicamente para el sistema que la contiene. Para que una imagen pueda ejecutarse en otros entornos --el equipo de un companero, un servidor de integracion continua o un cluster de produccion-- es necesario depositarla en un registro (registry), un servicio especializado en almacenar, versionar y distribuir imagenes de contenedores. Docker Hub es el registro publico por defecto que Docker asocia al ecosistema, y actua como el punto de partida natural para cualquier flujo de publicacion.

## Que es un registro de imagenes

Un registro de imagenes es un servicio cliente-servidor cuya responsabilidad principal es alojar imagenes de contenedores y servirlas bajo demanda mediante el protocolo definido en la especificacion OCI Distribution. Cuando se ejecuta `docker pull nginx:latest`, el cliente Docker contacta con el registro configurado (por defecto `docker.io`), negocia la autenticacion si es necesaria, descarga el manifiesto de la imagen y, a partir de ahi, obtiene cada una de las capas que la componen.

El flujo de descarga utiliza transferencia por capas: si una capa ya existe localmente porque otra imagen la comparte, Docker la reutiliza sin descargarla de nuevo. La comunicacion entre cliente y registro se realiza mediante HTTPS y emplea tokens de autenticacion bearer para controlar el acceso. El registro no solo almacena imagenes, sino que tambien gestiona etiquetas (tags), permite listar repositorios y, en implementaciones avanzadas, ofrece funciones de escaneo, firma y control de acceso detallado.

## Docker Hub: el registro publico por defecto

Docker Hub es el registro gestionado por Docker Inc. que viene preconfigurado en cualquier instalacion de Docker. Cuando se referencia una imagen sin prefijo de registro, Docker asume que pertenece a Docker Hub: `nginx:latest` es equivalente a `docker.io/library/nginx:latest`. La distincion es importante porque cualquier otro registro debe especificarse de forma explicita en la referencia de la imagen.

Docker Hub organiza las imagenes en repositorios, que pueden ser oficiales o de usuario. Los repositorios oficiales (`library/`) son mantenidos por Docker Inc. en colaboracion con los equipos de software originales y proporcionan imagenes de proyectos como Alpine, Ubuntu, Nginx o Python. Estos repositorios siguen unas practicas recomendadas de construccion, incluyen documentacion y reciben actualizaciones frecuentes.

Los repositorios de usuario siguen la convencion `usuario/repositorio:tag`. Cualquier cuenta gratuita en Docker Hub permite crear repositorios publicos ilimitados, mientras que los repositorios privados estan limitados a uno en el plan gratuito y requieren suscripcion de pago para ampliar ese limite.

## Anatomia de una referencia de imagen

La referencia completa de una imagen sigue la estructura:

```
[registro]/[usuario/]repositorio[:tag][@digest]
```

Cada componente tiene una funcion especifica:
- **registro**: direccion del servidor que aloja la imagen. Si se omite, se asume `docker.io`.
- **usuario**: espacio de nombres propietario del repositorio. En los repositorios oficiales es `library`.
- **repositorio**: nombre del conjunto de imagenes que conforman un proyecto.
- **tag**: etiqueta que identifica una version o variante concreta dentro del repositorio. Por defecto es `latest`.
- **digest**: hash SHA256 del manifiesto de la imagen, que proporciona una referencia inmutable.

```bash
# Referencias equivalentes
docker pull nginx:1.25
docker pull docker.io/library/nginx:1.25

# Referencia por digest (inmutable)
docker pull nginx@sha256:aabf3f3b8a9d1f23b5c8c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f
```

El digest es el unico identificador verdaderamente inmutable de una imagen. Mientras que una etiqueta puede reasignarse a una imagen diferente (un nuevo `push` con la misma tag sobrescribe la referencia), el digest vincula de forma inequivoca a un manifiesto concreto. En entornos de produccion donde la trazabilidad es critica, referenciar imagenes por digest elimina la ambiguedad de las etiquetas.

## Autenticacion y planes de Docker Hub

Docker Hub distingue entre acceso anonimo y autenticado. Sin autenticacion, Docker permite descargar imagenes publicas pero con limites de tasa (rate limits). Desde noviembre de 2020, las cuentas anonimas tienen un limite de 100 descargas cada seis horas, y las cuentas gratuitas autenticadas, de 200 descargas en el mismo periodo. Estos limites se aplican por direccion IP o por usuario, respectivamente.

```bash
# Iniciar sesion en Docker Hub
docker login

# Verificar el estado de autenticacion
cat ~/.docker/config.json
```

El comando `docker login` almacena las credenciales en `~/.docker/config.json` utilizando un token de acceso o contrasena. Docker no almacena la contrasena en texto plano, sino que genera un token de autenticacionBearer que se incluye en las cabeceras de las peticiones al registro. Para entornos automatizados, es recomendable utilizar tokens de acceso personal en lugar de la contrasena de la cuenta.

## Repositorios oficiales y su mantenimiento

Los repositorios oficiales son un sello de calidad dentro de Docker Hub. Para que un proyecto sea aceptado como oficial, debe cumplir criterios estrictos: seguir las mejores practicas de Dockerfile, proporcionar imagenes multiplataforma, actualizarse con frecuencia, incluir documentacion y someterse a revisiones de seguridad. Proyectos como Ubuntu, Node.js, Golang, Python y Nginx mantienen sus repositorios oficiales.

Estos repositorios utilizan convenciones de etiquetado estandarizadas. La etiqueta `latest` apunta siempre a la version estable mas reciente, pero existen variantes como `slim` (version reducida), `alpine` (basada en Alpine Linux para minimizar el tamano) o `bookworm`/`bullseye` (basadas en versiones especificas de Debian).

```bash
# La variante slim reduce significativamente el tamano
docker pull python:3.12-slim

# La variante alpine es la mas pequena
docker pull node:20-alpine
```

> Elegir la variante adecuada no es solo una cuestion de tamano. Las imagenes basadas en Alpine utilizan musl libc en lugar de glibc, lo que puede provocar incompatibilidades con algunas aplicaciones compiladas contra glibc. Siempre es recomendable probar la variante elegida en el entorno objetivo antes de llevarla a produccion.

## Limites y alternativas a Docker Hub

Docker Hub no es el unico registro disponible ni necesariamente el mas adecuado para todos los escenarios. Sus limites de tasa, la restriccion de repositorios privados en cuentas gratuitas y la dependencia de un servicio externo han impulsado la adopcion de alternativas. Otros registros publicos como GitHub Container Registry (ghcr.io), Quay.io (Red Hat) o GitLab Container Registry ofrecen modelos de precios distintos y se integran directamente con sus respectivas plataformas de CI/CD.

La decision de utilizar Docker Hub o un registro alternativo depende del flujo de trabajo: para proyectos open source o prototipos personales, Docker Hub gratuito es suficiente; para equipos que ya usan GitHub o GitLab, sus registros integrados eliminan la necesidad de gestionar credenciales separadas; para organizaciones con requisitos de compliance o residencia de datos, los registros privados on-premise son la unica opcion viable.

