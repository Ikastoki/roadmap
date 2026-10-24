# Seguridad en la distribucion y automatizacion con CI/CD

Distribuir imagenes no termina en el push. Cada imagen que viaja desde el entorno de construccion hasta el registro, y desde el registro hasta los nodos de ejecucion, atraviesa multiples puntos donde la integridad, la autenticidad y la seguridad pueden comprometerse. La firmacriptografica, el escaneo de vulnerabilidades y las politicas de etiquetado forman la capa de seguridad de la distribucion. La automatizacion mediante pipelines de CI/CD convierte estas practicas en procesos repetibles y auditable s.

## Docker Content Trust y firma de imagenes

Docker Content Trust (DCT) es un mecanismo que garantiza la integridad y la autenticidad de las imagenes mediante firmas criptograficas. Cuando DCT esta habilitado, el cliente Docker solo descarga o ejecuta imagenes que hayan sido firmadas por una clave privada de confianza. La tecnologia subyacente es Notary, un proyecto de CNCF que implementa el protocolo TUF (The Update Framework).

TUF disena un modelo de confianza en el que multiples roles pueden firmar distintos aspectos de un repositorio. El rol `targets` firma las referencias entre etiquetas y digests; el rol `snapshot` firma la lista completa de etiquetas; el rol `timestamp` firma la referencia mas reciente del snapshot; y el rol `root` firma las claves publicas de todos los demas roles. Esta separacion permite rotar claves comprometidas sin necesidad de volver a firmar todas las imagenes.

```bash
# Habilitar Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Al hacer push, Docker solicita la clave de firma y firma la imagen
docker push usuario/mi-app:1.0.0

# Verificar el estado de firma de una imagen
docker trust inspect usuario/mi-app:1.0.0
```

Cuando DCT esta activo, un `docker pull` falla si la imagen solicitada no tiene una firma valida. Esto evita que un atacante que haya conseguido acceso al registro pueda reemplazar una imagen legitima por una maliciosa sin romper la firma.

```bash
# Listar las claves de firma
docker trust key generate mi-clave

# Delegar firma a una clave remota
docker trust signer add --key mi-clave.pub usuario/mi-app usuario@miempresa.com
```

La gestion de las claves es el aspecto mas delicado de DCT. La clave raiz debe almacenarse offline, preferiblemente en un dispositivo HSM o al menos en un medio fisico no conectado a la red. Los equipos de desarrollo firman con claves delegadas que pueden rotarse sin invalidar la cadena de confianza.

## Escaneo de vulnerabilidades

Una imagen firmada no es necesariamente segura: la firma prueba que la imagen proviene de una fuente legitima, pero no dice nada sobre si su contenido contiene vulnerabilidades conocidas. El escaneo analiza las capas de la imagen, identifica los paquetes instalados y los cruza con bases de datos de vulnerabilidades como CVE, NVD o las bases propias del motor de escaneo.

Trivy, de Aqua Security, es uno de los escaneres mas utilizados por su rapidez, su cobertura de bases de datos (OS packages, librerias de lenguaje, dependencias de aplicaciones) y su facil integracion en pipelines.

```bash
# Escanear una imagen local con Trivy
trivy image usuario/mi-app:1.0.0

# Escanear y generar salida en formato JSON
trivy image --format json --output resultados.json usuario/mi-app:1.0.0

# Escanear solo vulnerabilidades de severidad CRITICAL y HIGH
trivy image --severity CRITICAL,HIGH usuario/mi-app:1.0.0
```

Snyk es otra alternativa que ofrece escaneo tanto en el pipeline como en el registro. A diferencia de Trivy, Snyk requiere una cuenta y un token de autenticacion, pero proporciona un panel centralizado con tendencias, priorizacion y remediacion.

Docker Scout es la solucion nativa de Docker. Se integra con Docker Desktop y Docker Hub, y proporciona analisis de vulnerabilidades junto con recomendaciones de versiones de paquetes que las corrigen.

```bash
# Analizar con Docker Scout
docker scout quickview usuario/mi-app:1.0.0

# Comparar la imagen con la version recomendada
docker scout recommendations usuario/mi-app:1.0.0
```

> El escaneo debe integrarse en el pipeline de CI/CD como una puerta de calidad. Si una imagen contiene vulnerabilidades de severidad critica y no hay una correccion disponible, el pipeline debe fallar y evitar que la imagen llegue al registro o, si ya esta en produccion, generar una alerta inmediata.

## Politicas de etiquetado

El etiquetado de imagenes no es una decision estetica; define la trazabilidad, la capacidad de rollback y la claridad del ciclo de release. Una politica de etiquetado bien disenada responde a tres preguntas: que version de la imagen se esta ejecutando, de que commit proviene y en que entorno deberia desplegarse.

El etiquetado con versionado semantico (SemVer) es el estandar mas extendido. Una etiqueta `v1.2.3` se compone de `MAJOR.MINOR.PATCH`, donde el incremento de MAJOR indica cambios incompatibles, MINOR anade funcionalidades compatibles hacia atras, y PATCH corrige errores.

```bash
# Etiquetas semanticas
docker tag mi-app:build-abc123 usuario/mi-app:1.2.3
docker tag mi-app:build-abc123 usuario/mi-app:1.2
docker tag mi-app:build-abc123 usuario/mi-app:1
```

La etiqueta `latest` es util en desarrollo por su comodidad, pero en produccion debe evitarse. `latest` es ambigua: no indica que version contiene, puede sobrescribirse en cualquier momento y rompe la trazabilidad. Si un despliegue referencia `latest` y alguien hace push de una nueva version, el siguiente despliegue obtendra una imagen diferente sin que nadie haya cambiado la configuracion explicitamente.

```bash
# En produccion: siempre referenciar por digest o version semantica
# Bien:
docker run usuario/mi-app:1.2.3

# Mal (ambiguo):
docker run usuario/mi-app:latest
```

Una practica recomendada es etiquetar cada imagen con tres referencias: la version semantica, el hash del commit de Git (o el identificador unico de build) y, opcionalmente, un alias movil como `latest` o `stable` para entornos no criticos.

```yaml
# Ejemplo de etiquetado en un pipeline
tags:
  - "1.2.3"
  - "1.2"
  - "1"
  - "sha-abc1234"
  - "latest"  # solo para entornos de desarrollo
```

## Automatizacion con CI/CD

La integracion continua y el despliegue continuo (CI/CD) son el vehiculo natural para automatizar la construccion, el escaneo, la firma y la publicacion de imagenes. Un pipeline tipico para imagenes Docker sigue esta secuencia: checkout del codigo, construccion de la imagen, escaneo de vulnerabilidades, firma (si se requiere), push al registro y, opcionalmente, despliegue en el entorno correspondiente.

### GitHub Actions

```yaml
name: Build and Push
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Login a GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Escanear con Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH

      - name: Build y push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
```

El pipeline hace checkout del repositorio, se autentica contra ghcr.io usando el token automatico de GitHub (`GITHUB_TOKEN`), escanea la imagen con Trivy (fallando si encuentra vulnerabilidades criticas o altas) y finalmente construye y sube la imagen con dos etiquetas: el SHA del commit y `latest`.

### GitLab CI

```yaml
variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

stages:
  - build
  - scan
  - push

build:
  stage: build
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker tag $DOCKER_IMAGE $CI_REGISTRY_IMAGE:latest

scan:
  stage: scan
  script:
    - trivy image --severity CRITICAL,HIGH --exit-code 1 $DOCKER_IMAGE

push:
  stage: push
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $DOCKER_IMAGE
    - docker push $CI_REGISTRY_IMAGE:latest
```

GitLab CI inyecta automaticamente las variables `CI_REGISTRY`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD` y `CI_REGISTRY_IMAGE` cuando el registro de GitLab esta habilitado en el proyecto. Esto elimina la necesidad de configurar credenciales manualmente en el pipeline.

## Proteccion del registro y buenas practices finales

Independientemente del registro utilizado, existen medidas de seguridad adicionales que deben aplicarse. El registro debe servirse exclusivamente sobre HTTPS con certificados validos. Las cuentas de servicio utilizadas en pipelines deben tener los minimos privilegios necesarios: acceso de solo lectura para despliegues y acceso de escritura restringido a un subconjunto de repositorios para las builds.

Las politicas de retencion de imagenes evitan la acumulacion de versiones antiguas que ya no se utilizan pero que ocupan espacio y, en algunos casos, contienen vulnerabilidades conocidas que los escaneres seguiran reportando. Tanto ECR como Harbor y ACR permiten definir reglas automaticas de expiracion.

Por ultimo, la inmutabilidad de las etiquetas (tag immutability) impide que una etiqueta existente se sobrescriba con una imagen diferente. Si un proceso de CI/CD intenta hacer push de una nueva version con una etiqueta ya existente, el registro lo rechaza. Esta politica fuerza a que cada cambio genere una nueva etiqueta, lo que garantiza que cualquier version desplegada en produccion pueda rastrearse hasta un commit concreto sin ambiguedad.

