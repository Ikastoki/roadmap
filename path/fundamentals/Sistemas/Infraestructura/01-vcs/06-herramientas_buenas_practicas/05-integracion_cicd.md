# Integracion con CI/CD

La integracion continua (CI) ejecuta pruebas, verificaciones de codigo y construccion automaticamente cada vez que se detectan cambios en el repositorio. El despliegue continuo (CD) automatiza la publicacion de esas construcciones en entornos de staging o produccion. Juntos, CI/CD eliminan la integracion manual, reducen errores y aceleran el ciclo de vida del software.

Git es el detonante de todo el pipeline: cada push, cada pull request, cada fusion puede activar una cadena de verificaciones que garantizan que el codigo cumple los estandares del proyecto antes de llegar a produccion.

## Como funciona CI/CD con Git

1. Un desarrollador hace push de sus cambios al repositorio remoto.
2. La plataforma de CI/CD detecta el evento y lanza un pipeline.
3. El pipeline ejecuta pasos definidos: instalar dependencias, correr tests, construir, analizar codigo, desplegar.
4. El resultado se reporta en la solicitud de fusion (pass/fail).
5. Si todo pasa, el codigo puede fusionarse con confianza.

## GitHub Actions

El sistema de CI/CD nativo de GitHub. Los pipelines se definen en archivos YAML en `.github/workflows/`.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Configurar Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar linter
        run: npm run lint

      - name: Ejecutar tests
        run: npm test -- --coverage

      - name: Construir proyecto
        run: npm run build

      - name: Subir reporte de cobertura
        uses: codecov/codecov-action@v3
```

## GitLab CI/CD

Los pipelines se definen en `.gitlab-ci.yml` en la raiz del repositorio.

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm run lint
    - npm test
  coverage: '/Statements\s*:\s*(\d+\.?\d*)%/'

build:
  stage: build
  image: node:20
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

deploy_staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  only:
    - develop
  when: manual
```

## Pull Request Checks

La caracteristica mas valiosa de CI/CD para la协作: los checks aparecen directamente en la solicitud de fusion. Los revisores pueden ver si los tests pasan, si la cobertura aumento o disminuyo, y si la construccion fue exitosa, antes de aprobar el cambio.

```
Checks: All checks passed
  ✓ test (node 18)      2m 15s
  ✓ test (node 20)      1m 58s
  ✓ test (node 22)      2m 03s
  ✓ build               1m 45s
  ✓ lint                0m 32s
```

## Reglas de proteccion con CI

Las plataformas permiten bloquear la fusion de PRs si los checks de CI fallan. Esto garantiza que nunca entre codigo que rompa la construccion o que no pase los tests.

**GitHub:** Settings > Branches > Branch protection rules > Require status checks to pass.

**GitLab:** Settings > Repository > Protected branches > Require approval from code owners.

## CI/CD mas alla de pruebas

Los pipelines tambien pueden ejecutar:

- **Analisis de seguridad:** escaneo de vulnerabilidades en dependencias (Snyk, Dependabot).
- **Despliegue automatico:** publicar la version mas reciente en staging o produccion.
- **Notificaciones:** alertar al equipo en Slack o Discord si algo falla.
- **Generacion de documentacion:** construir y publicar la documentacion del proyecto.
- **Release automatico:** crear tags y releases en GitHub/GitLab cuando se fusiona a main.

> CI/CD es la extension natural de los ganchos de Git al servidor. Mientras los hooks validan el codigo en la maquina del desarrollador, CI/CD valida en un entorno controlado y compartido. Juntos, eliminan la necesidad de "esperar a ver si funciona en produccion".

