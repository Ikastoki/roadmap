# CI/CD Integrado

Un despliegue manual de una aplicacion web sigue un proceso predecible: el desarrollador ejecuta tests, compila el codigo, sube los archivos al servidor, reinicia el servicio y verifica que funciona. Cada paso es una oportunidad para cometer un error: olvidar un archivo, subir la version incorrecta, reiniciar el servicio equivocado, o no verificar que la aplicacion levanto correctamente. La integracion continua y el despliegue continuo (CI/CD) eliminan estas oportunidades de error al automatizar toda la cadena: desde que un desarrollador hace push del codigo hasta que la aplicacion esta corriendo en produccion, pasando por compilacion, tests, despliegue y verificacion. El pipeline de CI/CD no es una herramienta sino un concepto: una secuencia de etapas automaticas que transforman codigo fuente en software funcionando.

## El concepto de pipeline de despliegue

Un pipeline de despliegue es una serie de etapas que se ejecutan en orden, donde cada etapa consume el resultado de la anterior. Si alguna etapa falla, el pipeline se detiene y notifica al equipo. Las etapas tipicas son:

1. **Compilacion**: el codigo fuente se transforma en artefactos ejecutables (binarios, containers, paquetes).
2. **Tests unitarios**: se verifica que cada componente individual funcione correctamente.
3. **Tests de integracion**: se verifica que los componentes funcionen juntos.
4. **Despliegue a staging**: los artefactos se instalan en un entorno que replica produccion.
5. **Tests de aceptacion**: se verifica que la aplicacion cumple con los requisitos funcionales.
6. **Despliegue a produccion**: los artefactos se instalan en los servidores de produccion.
7. **Verificacion post-despliegue**: se comprueba que la aplicacion esta operativa y respondiendo correctamente.

La separacion en etapas permite detectar problemas temprano. Un error en los tests unitarios se detecta en segundos, no despues de haber desplegado en produccion. Un problema de integracion se detecta antes de llegar a staging. Cada etapa es un filtro que evita que errores avancen hacia produccion.

## Herramientas de CI/CD

### GitHub Actions

GitHub Actions es un sistema de CI/CD integrado directamente en GitHub. Los pipelines se definen en archivos YAML dentro del directorio `.github/workflows/` del repositorio. Cada workflow se ejecuta en respuesta a eventos (push, pull request, schedule) y se ejecuta en contenedores virtuales provisionados por GitHub.

```yaml
name: Despliegue
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compilar
        run: make build
      - name: Ejecutar tests
        run: make test
      - name: Desplegar
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/miapp
            git pull origin main
            sudo systemctl restart miapp
```

Este workflow ejecuta compilar, tests y despliegue en cada push a la rama `main`. El paso de despliegue se conecta por SSH al servidor y ejecuta git pull mas reinicio del servicio. Los secrets (`SERVER_HOST`, `SSH_KEY`) se almacenan en la configuracion de GitHub y nunca se exponen en el codigo fuente.

### GitLab CI

GitLab CI funciona con un archivo `.gitlab-ci.yml` en la raiz del repositorio. Cada job se ejecuta en un **runner**: una maquina que ejecuta el pipeline. Los runners pueden ser compartidos (provisionados por GitLab) o dedicados (instalados en la infraestructura del equipo).

```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - make build
  artifacts:
    paths:
      - build/

test:
  stage: test
  script:
    - make test

deploy:
  stage: deploy
  only:
    - main
  script:
    - ssh deploy@server "cd /var/www/miapp && git pull && systemctl restart miapp"
```

La estructura por stages permite paralelizar jobs que no dependen entre si. `build` y `test` podrian ejecutarse simultaneamente si no hubiera dependencia entre ellos.

### Jenkins

Jenkins es una herramienta de CI/CD autoalojada, lo que significa que se instala en la infraestructura del equipo y se administra manualmente. Ofrece una interfaz web para configurar pipelines (mediante Jenkinsfile) y una amplia coleccion de plugins para integrarse con praticas casi cualquier herramienta. A diferencia de GitHub Actions y GitLab CI, Jenkins no esta acoplado a una plataforma de codigo fuente especifica.

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
        stage('Test') {
            steps {
                sh 'make test'
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sshagent(['server-key']) {
                    sh 'ssh deploy@server "cd /var/www/miapp && git pull && systemctl restart miapp"'
                }
            }
        }
    }
}
```

Jenkins usa Groovy como lenguaje para definir pipelines, lo cual ofrece flexibilidad logica (condicionales, bucles) que YAML no permite directamente. Sin embargo, esta flexibilidad puede hacer que los Jenkinsfiles sean mas complejos de mantener que los equivalentes en GitHub Actions o GitLab CI.

## Despliegue en servidores: los patrones

### Despliegue directo (in-place)

El patron mas simple: el servidor de produccion ejecuta git pull para obtener el codigo actualizado, instala dependencias y reinicia el servicio. Este enfoque es rapido pero tiene un problema critico: si el despliegue falla a mitad de camino, el servidor queda en un estado inconsistente. No hay forma rapida de volver a la version anterior.

### Blue-green deployment

Se mantienen dos copias del servidor (azul y verde). Mientras una sirve produccion, la otra se actualiza con la nueva version. Cuando la nueva version esta verificada, el balanceador de carga redirige el trafico a ella. Si hay problemas, se redirige de vuelta a la version anterior. Este patron elimina el downtime pero requiere duplicar infraestructura.

### Rolling update

En un cluster de servidores, se actualizan los nodos uno por uno. Mientras un nodo se actualiza, el trafico se redistribuye a los demas. Esto minimiza el impacto de un despliegue fallido pero requiere que la aplicacion soporte versiones multiples simultaneamente.

> Independientemente del patron elegido, el pipeline de CI/CD debe ejecutar verificaciones post-despliegue. Un despliegue que no verifica que la aplicacion funciona correctamente es un despliegue a ciegas.

