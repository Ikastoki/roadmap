# Valor añadido mas alla del VCS

Un repositorio Git solo almacena el historial de cambios del codigo. Las plataformas de colaboracion (GitHub, GitLab, Bitbucket, Gitea, Azure DevOps) añaden capas de funcionalidad que transforman un simple repositorio en un entorno completo de desarrollo: gestion de incidencias, revision de codigo, automatizacion de pruebas, despliegues, wikis y herramientas de gestion de proyecto. Sin estas capas, un equipo necesitaria multiples herramientas aisladas para coordinar el trabajo.

Estas plataformas son lo que diferencia a Git como herramienta local de Git como ecosistema de协作.

## Gestion de incidencias y tareas

Las incidencias (*issues*) son la unidad basica de trabajo en la plataforma. Cada incidencia representa una tarea, un error, una mejora o una pregunta que debe ser abordada.

- **Etiquetas** (*labels*): clasifican por tipo (bug, enhancement, documentation) o prioridad.
- **Milestones**: agrupan incidencias en hitos o versiones.
- **Asignacion**: indica que persona es responsable de resolver la incidencia.
- **Plantillas**: guian al reportero para incluir informacion relevante (pasos para reproducir, comportamiento esperado, entorno).

```markdown
<!-- Plantilla de incidencia -->
## Descripcion
[Descripcion clara del problema]

## Pasos para reproducir
1. Ir a '...'
2. Hacer clic en '...'
3. Ver error

## Comportamiento esperado
[Que deberia ocurrir]

## Entorno
- SO: [ej. Windows 11]
- Navegador: [ej. Chrome 120]
```

## Revision de codigo estructurada

Las Pull Requests (GitHub) o Merge Requests (GitLab) son la herramienta central de revision de codigo. Permiten que un desarrollador proponga cambios y que el equipo los revise antes de fusionarlos con la rama principal.

- **Comentarios inline**: observaciones sobre lineas especificas del codigo.
- **Aprobaciones**: formalizan que un revisor ha verificado la calidad del cambio.
- **Checks automaticos**: la plataforma ejecuta pruebas, linting y construccion automaticamente.
- **Reglas de protection**: bloquean la fusion si no hay aprobaciones suficientes o si los checks fallan.

## Automatizacion (CI/CD)

Las plataformas integran motores de CI/CD que ejecutan pipelines automaticamente cuando se detectan cambios en el repositorio.

```yaml
# Ejemplo: pipeline de GitHub Actions
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Instalar dependencias
        run: npm install
      - name: Ejecutar tests
        run: npm test
      - name: Build
        run: npm run build
```

## Wiki y documentacion del proyecto

Cada repositorio puede incluir una wiki que almacena documentacion complementaria: guias de instalacion, arquitectura, decisiones de diseno, FAQ. A diferencia de los archivos `README.md` en el repositorio, la wiki es un espacio separado para documentacion extensa.

## Tableros y gestion de proyecto

Las plataformas ofrecen tableros tipo Kanban que visualizan el flujo de trabajo: columnas por estado (por hacer, en progreso, revision, hecho) que se actualizan automaticamente al mover incidencias. Esto conecta directamente el codigo con la gestion del proyecto sin necesidad de herramientas externas.

> Las plataformas de colaboracion no reemplazan a Git: lo potencian. Un repositorio Git sin plataforma es como un libro sin editor: puedes escribir, pero no tienes revision, distribucion ni gestion editorial. Las plataformas añaden la capa social y organizativa que convierte el codigo en proyecto.

