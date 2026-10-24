# Claude Code CLI

## GitHub Actions

La accion `anthropics/claude-code-action` permite integrar Claude Code directamente en pipelines de GitHub Actions, transformando el agente en un asistente automatico de revision de codigo, correccion de errores y mantenimiento de repositorios. Esta integracion convierte a Claude Code en un participante activo del flujo CI/CD, capaz de analizar Pull Requests, ejecutar comandos arbitrarios y proponer cambios sin intervencion manual. El caso de uso mas inmediato es el code review automatico: cada PR desencadena una ejecucion de Claude Code que inspecciona el diff, identifica problemas de calidad, sugiere mejoras y, opcionalmente, aplica correcciones directamente.

La configuracion basica de la accion requiere especificar la version de Claude Code mediante el parametro `version` y los comandos a ejecutar a traves de `claude_args`. Este parametro acepta cualquier flag soportado por el CLI, lo que permite personalizar el comportamiento del agente en el contexto del pipeline. Por ejemplo, es posible pasar `--mcp-config` para cargar un conjunto de herramientas MCP especificas para el CI, o `--allowedTools` para restringir que operaciones puede realizar el agente durante la ejecucion automatizada, evitando riesgos de seguridad en un entorno que opera sin supervision humana.

```yaml
name: Claude Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          version: "latest"
          claude_args: >-
            --allowedTools Read,Write,Edit,Bash,Grep,Glob
            --mcp-config .github/mcp-ci.json
            --dangerously-skip-permissions
```

La generacion del token de autenticacion para entornos CI se realiza mediante `claude setup-token`. Este comando produce un token OAuth duradero que puede almacenarse como secreto de GitHub (`CLAUDE_CODE_TOKEN`) y ser consumido por la accion sin necesidad de interaccion manual. El token se genera una unica vez y se configura en el repositorio, permitiendo que todos los workflows subsecuentes automaticen sus operaciones con Claude Code sin exponer credenciales de usuario.

```bash
claude setup-token
```

> El flag `--dangerously-skip-permissions` desactiva el sistema de confirmacion de permisos, necesario en CI donde no hay un operador humano que autorice cada operacion. Debe combinarse con `--allowedTools` para establecer un perimetro de seguridad definido. Sin esta combinacion, Claude Code se detendria solicitando confirmacion en un entorno no interactivo.

El comando slash `/autofix-pr` automatiza la correccion de fallos detectados en CI. Cuando se invoca desde un comentario en un PR o como parte de un workflow, Claude Code analiza los logs de error, identifica la causa raiz y aplica las correcciones necesarias directamente en el codigo. Esto cubre desde errores de linting hasta fallos de compilacion o tests unitarios, reduciendo el ciclo de iteracion entre deteccion y solucion a minutos sin intervencion del desarrollador.

Para revisiones profundas en CI sin modo interactivo, `claude ultrareview` ejecuta un analisis exhaustivo de todo el codigo fuente del repositorio. A diferencia de una revision enfocada en el diff de un PR, `ultrareview` examina la totalidad del arbol fuente, identificando code smells, deuda tecnica, violaciones de patrones de diseno y oportunidades de refactorizacion. Su salida se formatea como comentarios directamente en el Pull Request, con referencias precisas a lineas de codigo y sugerencias accionables.

A continuacion se muestra un workflow completo de GitHub Actions que integra code review automatico, autofix de PR y autenticacion mediante token:

```yaml
name: Claude CI Pipeline
on:
  pull_request:
    types: [opened, synchronize, labeled]

jobs:
  code-review:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Claude Code Review
        uses: anthropics/claude-code-action@v1
        env:
          CLAUDE_CODE_TOKEN: ${{ secrets.CLAUDE_CODE_TOKEN }}
        with:
          version: "latest"
          claude_args: >
            --allowedTools Read,Write,Edit,Bash,Grep,Glob
            --dangerously-skip-permissions
            --pattern "aggressive"

      - name: Autofix PR on failure
        if: failure()
        uses: anthropics/claude-code-action@v1
        env:
          CLAUDE_CODE_TOKEN: ${{ secrets.CLAUDE_CODE_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          version: "latest"
          claude_args: >
            /autofix-pr
            --dangerously-skip-permissions
            --allowedTools Read,Write,Edit,Bash,Grep,Glob

      - name: Ultra Review on labeled PR
        if: contains(github.event.pull_request.labels.*.name, 'deep-review')
        uses: anthropics/claude-code-action@v1
        env:
          CLAUDE_CODE_TOKEN: ${{ secrets.CLAUDE_CODE_TOKEN }}
        with:
          version: "latest"
          claude_args: >
            ultrareview
            --dangerously-skip-permissions
            --allowedTools Read,Grep,Glob
```

La integracion con GitHub Actions no se limita a revisiones superficiales. `claude_args` admite cualquier combinacion de flags del CLI, incluyendo `--mcp-config` para cargar herramientas MCP especificas del entorno CI. Por ejemplo, un servidor MCP de base de datos puede ejecutar consultas de validacion de esquemas, o un servidor de testing puede lanzar suites completas y analizar los resultados. El control granular mediante `--allowedTools` permite disenar politicas restrictivas: en un workflow de revision se pueden limitar las herramientas a solo lectura (`Read, Grep, Glob`), mientras que en un workflow de autofix se habilitan tambien `Write, Edit, Bash`. Esta separacion de privilegios es fundamental para mantener la seguridad en pipelines automatizados donde el agente opera con acceso al repositorio.

## Remote Control

`claude remote-control` inicia un servidor HTTP local que expone una interfaz de control remoto para Claude Code, accesible desde claude.ai o la aplicacion movil de Claude. Este mecanismo permite a un desarrollador iniciar, supervisar y gestionar sesiones de Claude Code desde cualquier dispositivo, sin necesidad de estar frente al terminal donde se ejecuta el agente. El servidor actua como un puente bidireccional: recibe instrucciones desde el exterior y reenvia la salida de Claude Code en tiempo real.

El flag `--name` asigna un identificador descriptivo al proyecto, que aparece en la interfaz de claude.ai para distinguir multiples sesiones remotas. Los flags `--port` y `--hostname` configuran la direccion de escucha del servidor, permitiendo exponerlo en una interfaz de red especifica o en un puerto concreto. Por defecto, el servidor escucha en `localhost:8080`, pero en entornos de produccion o equipos distribuidos puede ser necesario vincularlo a `0.0.0.0` o a una IP interna de la red corporativa.

```bash
claude remote-control --name "backend-api" --port 9090 --hostname 0.0.0.0
```

Los casos de uso practicos del control remoto abarcan escenarios donde el terminal principal no es accesible. Un desarrollador puede iniciar una tarea de refactorizacion desde el movil mientras se desplaza, monitorizar el progreso de una migracion de base de datos desde claude.ai en el navegador, o revisar los resultados de un analisis de seguridad que se ejecuta en un servidor de desarrollo remoto. La sesion permanece activa en la maquina donde se lanzo `claude remote-control`, pero la interaccion se realiza desde el dispositivo conectado.

La combinacion de GitHub Actions para automatizacion en CI/CD y Remote Control para supervision remota convierte a Claude Code en una plataforma de desarrollo accesible desde cualquier lugar y automatizable en cualquier etapa del pipeline. El programador conserva el control granular de permisos y herramientas en cada contexto, mientras que el agente ejecuta las tareas rutinarias de revision, correccion y mantenimiento de forma autonomia.

