# Claude Code CLI

## Plan mode vs Build mode

Claude Code distingue dos modos de operacion que definen el ambito de accion del agente y que conviene comprender como fases complementarias de un mismo flujo de trabajo, no como alternativas excluyentes. El modo Plan (`claude --mode plan` o el comando `/plan` dentro de una sesion) restringe al agente a operaciones de solo lectura: puede leer archivos, navegar por el arbol de directorios, ejecutar busquedas y analizar el codigo, pero no tiene capacidad de modificarlo ni de ejecutar comandos que alteren el sistema. Este modo esta disenado para las fases de analisis y diseno, donde el objetivo es entender la arquitectura existente, identificar problemas, evaluar enfoques y producir una especificacion detallada de los cambios propuestos. El modo Build, por el contrario, concede acceso completo a todas las herramientas: escritura, edicion, ejecucion de comandos, interaccion con Git y cualquier otra operacion que requiera modificar el proyecto.

```bash
# Iniciar sesion en modo plan
claude --mode plan

# Cambiar a modo plan durante una sesion interactiva
/plan

# Iniciar directamente en modo build (por defecto)
claude
```

El flujo recomendado consiste en planificar primero y construir despues. Una sesion tipica comienza en modo Plan, donde se exploran los ficheros relevantes, se discuten las alternativas y se produce una especificacion detallada de los cambios necesarios. Una vez que el plan esta consensuado, se transpasa al modo Build, que ejecuta las modificaciones punto por punto siguiendo la especificacion. Esta separacion reduce drastricamente los errores por accion prematura y permite que el desarrollador mantenga el control conceptual sobre el que y el como. El modo Plan produce descripciones estructuradas que el modo Build interpreta como una hoja de ruta, evitando desviaciones y asegurando que cada cambio responde a una decision previamente validada.

> `/plan` no es un simple bloqueo de escritura: genera automaticamente una especificacion detallada de los cambios, estructurada por archivos, que puede revisarse, discutirse y refinarse antes de ejecutarla. Esa especificacion queda registrada en el historial de la sesion como artefacto de diseno.

## Sesiones efectivas

La calidad del resultado que produce Claude Code depende en gran medida de la calidad del prompt inicial. Un prompt efectivo incluye tres elementos: el contexto necesario para que el agente comprenda el proyecto y la situacion actual, el objetivo concreto que debe alcanzarse y las restricciones relevantes que condicionan la solucion. Proporcionar estos elementos de forma explicita al inicio de la sesion evita que el agente tenga que deducirlos del arbol de directorios o del contenido de los archivos, ahorrando tiempo y tokens. Referenciar archivos relevantes con la tecla de espacio o mediante mencion directa en el prompt permite al agente cargar el contenido en contexto sin necesidad de navegar manualmente.

```bash
# Prompt efectivo: contexto, objetivo y restricciones
claude "Estamos en el modulo de autenticacion (src/auth/). Necesito anyadir
soporte para refresh tokens usando JWT. La solucion debe ser compatible
con la base de datos PostgreSQL existente y mantener la compatibilidad
con los tokens actuales."
```

El Side Chat es una funcionalidad que permite hacer preguntas al agente sin interrumpir el flujo de trabajo principal. En la aplicacion de escritorio se activa con `Cmd+;` (macOS) o `Ctrl+;` (Windows/Linux), y en la CLI se invoca con `--btw` o `/btw`, que abre un canal de comunicacion paralelo donde el agente responde sin alterar el estado de la sesion principal. Esta separacion es util para consultas rapidas sobre documentacion, dudas sintacticas o verificaciones que no deberian contaminar el contexto de la tarea en curso.

La gestion del contexto es un aspecto critico en sesiones prolongadas. A medida que la conversacion avanza, el modelo acumula informacion que puede saturar la ventana de contexto disponible. Claude Code ofrece dos comandos para mitigar este problema: `/compact` comprime la conversacion eliminando los detalles redundantes y conservando solo la informacion esencial, y `/context` muestra una visualizacion del uso actual del contexto, desglosado por archivos cargados, comandos ejecutados y turnos de dialogo. Adicionalmente, las sesiones nombradas con el flag `--resume` permiten mantener un historial separado por feature branch, de modo que cada rama de desarrollo tenga su propia linea temporal conversacional sin interferencias cruzadas.

```bash
# Iniciar sesion nombrada para una feature especifica
claude --resume feature/refresh-tokens

# Consulta sin interrumpir la sesion principal (Side Chat)
/btw "Cual es la sintaxis de async/await en Python 3.11?"

# Compactar contexto cuando la conversacion se alarga
/compact

# Visualizar el uso de contexto
/context
```

> El Side Chat es especialmente util cuando se trabaja con subagentes: se puede preguntar al agente principal por el progreso de una tarea delegada sin necesidad de cambiar de contexto o interrumpir la ejecucion del subagente.

## Seguridad

El sistema de permisos de Claude Code constituye la primera linea de defensa contra operaciones no deseadas. La interfaz TUI muestra cada comando antes de ejecutarlo, con el contenido completo y los argumentos expandidos, permitiendo al usuario revisar visualmente que es exactamente lo que el agente va a hacer. Este es el momento critico para la intervencion manual: cualquier comando sospechoso, cualquier ruta inesperada o cualquier operacion que no se haya solicitado explicitamente debe detenerse en este punto.

```bash
# Diagnostico de configuracion y permisos
claude doctor

# Verificar la integridad de la instalacion
claude doctor --verbose
```

La eleccion del modo de permiso adecuado para cada tarea es una practica de seguridad basica. El modo `plan` debe utilizarse siempre que se este explorando codigo sin intencion de modificarlo, pues ni siquiera el modo permisivo puede alterar archivos en este estado. El modo `auto` es el apropiado para el trabajo cotidiano: el clasificador integrado evalua cada operacion y solo solicita confirmacion cuando detecta algo potencialmente peligroso, como la eliminacion de archivos o la ejecucion de comandos con efectos de red. El modo `bypassPermissions` (o su alias `--dangerously-skip-permissions`) suprime todas las confirmaciones y debe reservarse exclusivamente para entornos de integracion continua, contenedores efimeros o sesiones donde se ha verificado manualmente que el agente no tiene acceso a recursos sensibles.

Los datos locales de configuracion y cache de Claude Code residen en `~/.claude/`, que contiene las claves de API, las preferencias de modelo y el historial de sesiones. Este directorio no debe incluirse en repositorios Git ni compartirse entre equipos. El archivo `CLAUDE.md` es el mecanismo para definir reglas y comportamiento del agente a nivel de proyecto, almacenado en la raiz del repositorio y versionado con el codigo. Es importante recordar que `CLAUDE.md` es un archivo publico dentro del repositorio: nunca deben incluirse en el claves de API, tokens de acceso, URLs de servicios internos o cualquier otro secreto.

```bash
# La configuracion local del usuario no debe versionarse
echo "~/.claude/" >> .gitignore
```

> El comando `claude doctor` diagnostica problemas de configuracion, permisos incorrectos en el sistema de archivos, conflictos de versiones y errores de autenticacion. Es la primera herramienta que debe ejecutarse ante cualquier comportamiento anomalo del agente.

## Hooks y automatizacion

Claude Code expone dos puntos de extension fundamentales para automatizar flujos de trabajo: `PreToolUse` y `PostToolUse`. Estos hooks se definen en `CLAUDE.md` y se ejecutan automaticamente antes y despues de cada invocacion de herramienta, respectivamente. Un caso de uso tipico para `PreToolUse` es la ejecucion de un linter antes de cualquier operacion de escritura, asegurando que el codigo generado cumple con los estandares del proyecto antes de persistirse en disco. Si el linter falla, el hook puede cancelar la operacion o notificar al agente para que corrija el problema antes de continuar.

```markdown
<!-- CLAUDE.md -->
## Hooks

PreToolUse:
  - Si la herramienta es Write o Edit, ejecutar `npx eslint --quiet <ruta>`
    antes de permitir la escritura.
  - Si el linter falla, notificar al agente y cancelar la operacion.

PostToolUse:
  - Despues de cada operacion Write, ejecutar `npm run typecheck` para
    verificar que no se introducen errores de tipos.
  - Si la verificacion falla, registrar el error pero no bloquear.
```

El hook `PostToolUse` se ejecuta inmediatamente despues de que una herramienta completa su ejecucion, con acceso al resultado de la misma. Es el lugar adecuado para enviar notificaciones a servicios externos, registrar metricas de uso, actualizar tableros de seguimiento o disparar pipelines de CI. Por ejemplo, un equipo podria configurar un `PostToolUse` que notifique a un canal de Slack cada vez que Claude Code modifica un archivo de produccion, o que registre en una base de datos local las estadisticas de cada sesion para su posterior analisis.

La integracion con herramientas CLI externas es una de las estrategias mas efectivas para extender las capacidades de Claude Code sin consumir tokens en llamadas a APIs REST. Herramientas como `gh` (GitHub CLI), `aws` (AWS CLI), `gcloud` (Google Cloud CLI) o `sentry-cli` ejecutan operaciones complejas con una unica llamada al modelo, que simplemente construye el comando y procesa la salida. Este enfoque es drasticamente mas eficiente que invocar APIs REST directamente, donde cada peticion requiere que el modelo genere headers HTTP, maneje autenticacion, procese JSON de respuesta y gestione errores de red.

```bash
# Usar gh CLI en lugar de la API REST de GitHub
# En vez de llamar a la API de GitHub directamente...
# ... Claude Code ejecuta:
gh pr view 42 --json title,body,additions,deletions

# Usar sentry-cli para consultar errores
sentry-cli events list --project mi-proyecto --limit 5

# Usar aws CLI para desplegar infrastructure
aws lambda update-function-code --function-name mi-funcion \
  --zip-file fileb://deploy.zip
```

La combinacion de hooks, permisos granulares y herramientas CLI externas permite construir flujos de automatizacion sofisticados donde Claude Code actua como orquestador. Un equipo puede configurar un pipeline que ejecute el linter antes de cada escritura, verifique los tipos despues de cada cambio, despliegue automaticamente en un entorno de pruebas si todas las validaciones pasan y notifique el resultado a traves de Slack, todo ello sin que el desarrollador tenga que intervenir manualmente en cada paso. La clave para que estos flujos sean mantenibles es mantener la configuracion en `CLAUDE.md`, donde es visible, versionable y modificable por cualquier miembro del equipo.

