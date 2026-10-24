# Docker Hub

Docker Hub es el registro publico de imagenes de contenedores gestionado por Docker Inc. Funciona como repositorio centralizado donde desarrolladores y organizaciones publican, comparten y descargan imagenes. Su funcion en el ecosistema Docker es analoga a la de npm para Node.js o PyPI para Python: actua como la fuente de distribucion por defecto desde la que Docker extrae imagenes cuando no se especifica un registro alternativo.

## Busqueda de imagenes

El comando `docker search` permite explorar imagenes disponibles en Docker Hub directamente desde la terminal. Los resultados incluyen el nombre, la descripcion, el numero de estrellas y la calificacion oficial o automatizada de cada imagen.

```bash
# Buscar imagenes relacionadas con nginx
docker search nginx

NAME                         DESCRIPTION                      STARS     OFFICIAL
nginx                        Official build of Nginx          20000     [OK]
nginxinc/nginx               Official NGINX Docker Image      1800
nginx/nginx-ingress          NGINX Ingress Controller         1200
nginx/unit                  NGINX Unit - universal web...    300
```

Los filtros permiten refinar la busqueda para mostrar solo imagenes oficiales o con un minimo de estrellas:

```bash
# Buscar solo imagenes oficiales
docker search nginx --filter "is-official=true"

# Buscar con un minimo de estrellas
docker search nginx --filter "stars=1000"
```

Aunque `docker search` es util para exploracion rapida, la mayoria de los usuarios prefieren consultar Docker Hub directamente desde el navegador en `hub.docker.com`, donde la interfaz grafica ofrece informacion mas detallada: tags disponibles, capas, instrucciones de uso, enlaces a la documentacion y al repositorio fuente.

## Imagenes oficiales vs imagenes de la comunidad

Docker Hub clasifica las imagenes en dos categorias principales que determinan su nivel de confianza y mantenimiento.

Las imagenes oficiales son mantenidas por Docker Inc. en colaboracion con los equipos upstream de cada proyecto. Cumplen con practicas recomendadas de seguridad, reciben actualizaciones regulares y pasan por un proceso de revision. Entre las imagenes oficiales mas utilizadas se encuentran `nginx`, `ubuntu`, `python`, `node`, `alpine`, `postgres` y `redis`.

```bash
# Las imagenes oficiales no llevan prefijo de usuario
docker pull nginx       # oficial
docker pull ubuntu      # oficial
docker pull node        # oficial

# Las imagenes de comunidad llevan prefijo de usuario o cuenta
docker pull bitnami/nginx    # mantenida por Bitnami
docker pull jwilder/nginx-proxy  # mantenida por un usuario
```

Las imagenes de comunidad pueden ser mantenidas por organizaciones reconocidas (Bitnami, CircleCI, IBM) o por usuarios individuales. Aunque muchas son de alta calidad, el usuario debe evaluar su fiabilidad revisando las estrellas, la frecuencia de actualizacion, la documentacion y, preferiblemente, el Dockerfile publicado en un repositorio publico.

> El riesgo de utilizar imagenes no verificadas es real: se han documentado casos de imagenes maliciosas en Docker Hub que exfiltraban credenciales o incluian mineros de criptomonedas. Siempre que exista una version oficial, es preferible usarla. Si no existe, revisar el Dockerfile y las estrellas antes de confiar en una imagen de comunidad.

## Autenticacion y limites de descarga

Docker Hub impone limites de descarga (rate limits) que dependen del plan del usuario. Los usuarios anonimos tienen un limite de 100 descargas cada 6 horas por direccion IP, mientras que los usuarios autenticados (plan gratuito) tienen 200 descargas cada 6 horas. Los planes de pago eliminan o amplian estos limites.

```bash
# Iniciar sesion en Docker Hub
docker login

# Verificar el estado de autenticacion
docker system info | grep -i "registry"

# Cerrar sesion
docker logout
```

Estos limites afectan especialmente a entornos CI/CD donde multiples builds descargan las mismas imagenes repetidamente. La estrategia recomendada es autenticarse con una cuenta de Docker Hub para duplicar el limite y, en entornos empresariales, utilizar un mirror de registro local (registry mirror) o un registro privado.

## Repositorios y automatizacion

Cada imagen en Docker Hub se organiza en repositorios que pueden ser publicos o privados. Los repositorios publicos son gratuitos e ilimitados; los privados estan limitados en el plan gratuito a un numero reducido. Los repositorios pueden conectarse a repositorios de codigo fuente (GitHub, GitLab, Bitbucket) para construir imagenes automaticamente mediante builds automatizados cada vez que se empuja un cambio al repositorio.

```bash
# Etiquetar y subir una imagen al repositorio personal
docker tag mi-app:latest usuario/mi-app:latest
docker push usuario/mi-app:latest

# Subir con version semantica
docker tag mi-app:latest usuario/mi-app:1.0.0
docker push usuario/mi-app:1.0.0
```

> Aunque Docker Hub es el registro por defecto, no es la unica opcion. GitHub Container Registry (ghcr.io), Amazon ECR, Google Artifact Registry, Azure Container Registry y registros auto-gestionados con el componente Open Source `registry:2` son alternativas habituales en entornos profesionales. Para utilizar un registro distinto, las imagenes se etiquetan anteponiendo la URL del registro: `docker tag mi-app ghcr.io/usuario/mi-app:latest`.

