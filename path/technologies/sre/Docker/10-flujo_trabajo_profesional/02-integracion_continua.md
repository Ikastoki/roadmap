# Docker en la integracion continua

La integracion continua (CI) se beneficia de Docker porque cada ejecucion de pipeline arranca desde un estado conocido y reproducible. No importa si el agente de CI se ha reutilizado para otras comprobaciones: el contenedor garantiza un entorno limpio, aislado y desechable. Cada commit dispara el mismo proceso de build y test, eliminando la variabilidad entre ejecuciones.

## Tests en contenedores limpios

Ejecutar los tests dentro de un contenedor ofrece la garantia de que las dependencias coinciden exactamente con las definidas en el Dockerfile. El flujo tipico consiste en construir una imagen dedicada para tests y lanzar los contenedores necesarios.

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine

    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: |
          docker build -t myapp:test --target test .
          docker run --rm \
            -e DATABASE_URL=postgres://postgres:test@postgres:5432/test \
            -e REDIS_URL=redis://redis:6379 \
            --network ci \
            myapp:test
```

```yaml
# .gitlab-ci.yml
variables:
  DOCKER_DRIVER: overlay2

services:
  - postgres:16
  - redis:7-alpine

before_script:
  - docker info

test:
  script:
    - docker build -t myapp:test --target test .
    - docker run --rm myapp:test pytest --junitxml=report.xml
  artifacts:
    reports:
      junit: report.xml
```

> Elegir la imagen base adecuada para tests marca la diferencia. Las imagenes `-slim` reducen el tiempo de descarga, mientras que `-alpine` minimiza la superficie de ataque. Si el rendimiento de compilacion es critico, las imagenes basadas en `ubuntu` ofrecen mejor compatibilidad con binarios precompilados.

El uso de `docker build --target` permite construir solo la etapa de tests sin incluir las capas de produccion, acelerando el pipeline.

```dockerfile
FROM node:20 AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS test
COPY . .
RUN npm test

FROM base AS production
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

## Construccion y etiquetado automatico

Cada build en CI debe producir una imagen etiquetada de forma que sea posible rastrear su origen. El esquema de etiquetado mas robusto combina el identificador del commit con referencias semanticas.

```bash
# Variables disponibles en CI
COMMIT_SHA=$(git rev-parse --short HEAD)
BRANCH_NAME=$(git rev-parse --abbrev-ref HEAD | sed 's/[^a-zA-Z0-9._-]/-/g')

# Etiquetado multiple
docker build -t registry.example.com/myapp:$COMMIT_SHA \
             -t registry.example.com/myapp:$BRANCH_NAME \
             -t registry.example.com/myapp:latest .

docker push --all-tags registry.example.com/myapp
```

Para releases, el etiquetado semantico permite identificar versiones concretas:

```bash
# En un tag de version (v1.2.3)
VERSION=$(git describe --tags --abbrev=0 | sed 's/^v//')
docker build -t registry.example.com/myapp:$VERSION \
             -t registry.example.com/myapp:latest .
docker push registry.example.com/myapp:$VERSION
docker push registry.example.com/myapp:latest
```

Las etiquetas `latest`, `stable` y `edge` son convenciones utiles pero requieren disciplina: `latest` siempre apunta al ultimo commit de la rama principal, `stable` a la ultima version estable y `edge` al ultimo build de una rama de desarrollo.

## Cache de capas en CI

La principal optimizacion en pipelines de Docker es la reutilizacion de la cache de capas. Sin ella, cada build descargaria e instalaria todas las dependencias desde cero.

```bash
# Pull de la imagen previa para usar como cache
docker pull registry.example.com/myapp:latest || true

docker build --cache-from registry.example.com/myapp:latest \
             -t registry.example.com/myapp:$COMMIT_SHA .
```

En sistemas como GitLab CI con Docker-in-Docker, la cache se conserva entre ejecuciones del mismo runner. GitHub Actions utiliza `docker/build-push-action` que maneja la cache automaticamente:

```yaml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: registry.example.com/myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

> El modo `max` en `cache-to` almacena todas las capas, no solo las finales, lo que optimiza los builds de ramas con Dockerfiles muy divergentes, aunque consume mas espacio de cache.

## Push a registros y autenticacion

Tras construir y etiquetar, las imagenes deben enviarse a un registro accesible desde los entornos de despliegue. La autenticacion se configura mediante tokens o credenciales gestionadas por el proveedor de CI.

```bash
# Login con token
echo $REGISTRY_TOKEN | docker login registry.example.com -u $REGISTRY_USER --password-stdin

# Push de multiples tags
docker push registry.example.com/myapp:$COMMIT_SHA
docker push registry.example.com/myapp:$BRANCH_NAME
```

Los registros principales son Docker Hub, GitHub Container Registry (ghcr.io), GitLab Container Registry, Amazon ECR, Google Artifact Registry y Azure Container Registry. La eleccion depende del ecosistema del proyecto. Es recomendable utilizar un solo registro por organizacion para simplificar la gestion de accesos y politicas.

## Pipelines completos: build, test, push, scan

Un pipeline maduro encadena varias fases que garantizan la calidad de la imagen antes de su publicacion:

```yaml
stages:
  - test
  - build
  - scan
  - push

unit-test:
  stage: test
  script:
    - docker build --target test -t myapp:test .
    - docker run myapp:test

build:
  stage: build
  script:
    - docker build -t registry.example.com/myapp:$CI_COMMIT_SHA .

security-scan:
  stage: scan
  script:
    - docker scout analyze registry.example.com/myapp:$CI_COMMIT_SHA
    - trivy image --severity HIGH,CRITICAL registry.example.com/myapp:$CI_COMMIT_SHA

push:
  stage: push
  script:
    - docker tag registry.example.com/myapp:$CI_COMMIT_SHA registry.example.com/myapp:latest
    - docker push --all-tags registry.example.com/myapp
  only:
    - main
```

La fase de escaneo evalua vulnerabilidades en las capas de la imagen. Herramientas como Docker Scout, Trivy, Snyk o Grype se integran directamente en el pipeline y pueden bloquear el push si se supera un umbral de criticidad.

## Testing de infraestructura con Container Structure Test

Ademas de los tests de la aplicacion, es posible validar la propia imagen con herramientas como `container-structure-test`:

```yaml
- name: Structure test
  run: |
    container-structure-test test \
      --image registry.example.com/myapp:${{ github.sha }} \
      --config structure-test.yaml
```

```yaml
# structure-test.yaml
schemaVersion: "2.0.0"
commandTests:
  - name: "Python version"
    command: "python"
    args: ["--version"]
    expectedOutput: ["3.11.*"]
fileExistenceTests:
  - name: "Entrypoint"
    path: "/entrypoint.sh"
    shouldExist: true
```

Estas comprobaciones verifican que la imagen contiene los binarios, configuraciones y permisos esperados, actuando como una capa adicional de validacion antes del despliegue.

