# CI/CD y registries

La integracion continua y el despliegue continuo automatizan el flujo desde el codigo hasta la imagen desplegada. El pipeline construye la imagen, ejecuta pruebas, escanea vulnerabilidades, etiqueta y publica en un registry, todo sin intervencion manual.

## Flujo tipico

```
Codigo ──► Build ──► Test ──► Scan ──► Tag ──► Push ──► Deploy
```

## GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/mi-app:${{ github.sha }}
            ${{ secrets.DOCKER_USERNAME }}/mi-app:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ secrets.DOCKER_USERNAME }}/mi-app:${{ github.sha }}
          format: table
          exit-code: 1
          severity: HIGH,CRITICAL
```

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - push

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE .
    - docker push $IMAGE

test:
  stage: test
  script:
    - docker run --rm $IMAGE npm test

scan:
  stage: scan
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE

push:
  stage: push
  script:
    - docker tag $IMAGE $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

## Jenkins

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t mi-app:${BUILD_NUMBER} .'
            }
        }
        stage('Test') {
            steps {
                sh 'docker run --rm mi-app:${BUILD_NUMBER} npm test'
            }
        }
        stage('Scan') {
            steps {
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL mi-app:${BUILD_NUMBER}'
            }
        }
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh 'docker tag mi-app:${BUILD_NUMBER} $USER/mi-app:${BUILD_NUMBER}'
                    sh 'docker push $USER/mi-app:${BUILD_NUMBER}'
                }
            }
        }
    }
}
```

## Despliegue automatico

```yaml
# Docker Compose con imagen actualizada
services:
  app:
    image: miusuario/mi-app:${APP_VERSION:-latest}
    # Actualizar APP_VERSION en el pipeline para desplegar
```

```bash
# Desplegar nueva version
APP_VERSION=v1.2.3 docker compose up -d
```

> La automatizacion del pipeline de CI/CD elimina el error humano y garantiza que cada imagen publicada haya pasado por las mismas pruebas y escaneos de seguridad. La combinacion de build automatico, pruebas, escaneo de vulnerabilidades y publicacion crea un flujo de trabajo que es rapido, fiable y auditable.

