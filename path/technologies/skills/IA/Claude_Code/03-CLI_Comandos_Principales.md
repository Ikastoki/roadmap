# Claude Code CLI

La linea de comandos de Claude Code es la puerta de entrada a toda su funcionalidad. Aunque el modo interactivo TUI sea la cara mas visible, el CLI expone un abanico de subcomandos y flags que permiten desde ejecuciones puntuales no interactivas hasta la gestion de sesiones background, pasando por administracion de plugins, servidores MCP y gateways self-hosted. Dominar estos comandos es lo que diferencia un uso ocasional de un flujo de trabajo integrado en el dia a dia del desarrollo.

## Uso basico

La forma mas directa de usar Claude Code es simplemente invocar `claude` sin argumentos, lo que arranca una sesion interactiva en el directorio actual. Al hacerlo, Claude descubre automaticamente el contexto del proyecto: busca un archivo `CLAUDE.md` en la raiz para obtener instrucciones especificas, analiza la estructura de directorios para comprender la arquitectura, y examina el historial de git para entender el estado del repositorio. Si se prefiere iniciar la sesion con un prompt concreto, se pasa como argumento posicional:

```bash
claude
claude "Refactoriza el modulo de autenticacion para usar tokens JWT"
```

Para ejecuciones no interactivas, donde Claude debe responder y terminar, se usa la flag `-p` (o `--print`). Este modo es ideal para scripting y tuberias Unix: el modelo procesa la consulta, imprime la respuesta por stdout y finaliza, sin entrar en el bucle de dialogo. El contenido de entrada puede venir de un argumento, de un archivo, o de una tuberia, lo que permite encadenar Claude Code con otras herramientas del ecosistema Unix:

```bash
claude -p "Explica este codigo" < archivo.py
cat archivo.log | claude -p "Resume los errores principales"
```

La continuidad entre sesiones se maneja con las flags `-c` y `-r`. `claude -c` reanuda la conversacion mas reciente, recuperando todo el contexto previo. `claude -r "<id_o_nombre>" "query"` permite reanudar una sesion concreta por su identificador unico o por el nombre asignado durante su creacion. Esto es util cuando se trabaja en multiples temas de forma concurrente y se necesita retomar exactamente donde se dejo cada uno.

```bash
claude -c
claude -r "refactor-auth" "Continua con el siguiente modulo"
```

## Background y sesiones

Una de las capacidades mas potentes de Claude Code es la ejecucion en background. Con `claude --bg "tarea"` se inicia una sesion que opera de forma autonomia mientras el terminal queda libre para seguir trabajando. Claude ejecuta la tarea, navega por el arbol de directorios, modifica archivos y ejecuta comandos sin ocupar la terminal. El control se devuelve inmediatamente y la sesion queda registrada con un identificador unico.

```bash
claude --bg "Actualiza todas las dependencias a sus ultimas versiones estables"
```

Para monitorizar las sesiones activas se usa `claude agents`, que abre el Agent View, una interfaz dentro del TUI que muestra el estado de cada agente background. Cuando se necesita integrar la supervision en scripts, `claude agents --json` lista las sesiones activas en formato JSON, incluyendo IDs, nombres, estados y timestamps, facilitando su procesamiento programatico.

```bash
claude agents
claude agents --json | jq '.[] | {id: .id, status: .status}'
```

La interaccion con sesiones individuales se realiza mediante varios subcomandos. `claude attach <id>` adjunta la terminal actual a una sesion background, permitiendo ver su output en tiempo real o incluso intervenir si es necesario. `claude stop <id>` y `claude kill <id>` detienen la sesion de forma controlada o forzada respectivamente, mientras que `claude rm <id>` la elimina de la lista de sesiones registradas. Para inspeccionar el trabajo realizado sin adjuntarse, `claude logs <id>` imprime el output mas reciente de la sesion, y `claude respawn <id>` la reinicia desde cero si fallo o si se necesita repetir la ejecucion.

```bash
claude attach "sesion-abc123"
claude logs "sesion-abc123"
claude stop "sesion-abc123"
```

Todas las sesiones background dependen de un proceso supervisor llamado daemon. `claude daemon status` comprueba si el daemon esta activo, su tiempo de ejecucion y el numero de sesiones gestionadas. `claude daemon stop` lo detiene de forma limpia, lo que finaliza todas las sesiones background activas. Es recomendable verificar el estado del daemon si se sospecha que las sesiones background no estan respondiendo.

```bash
claude daemon status
claude daemon stop
```

## Otros comandos

La gestion de la extension funcional de Claude Code se canaliza a traves de `claude mcp` y `claude plugin`. El subcomando `claude mcp` configura servidores del Model Context Protocol, permitiendo anadir, listar, autenticar y depurar conexiones con herramientas externas como sistemas de monitorizacion, bases de datos o APIs de terceros. Por su parte, `claude plugin` administra plugins que extienden las capacidades del agente, ya sean instalados desde npm o locales.

```bash
claude mcp add
claude plugin @anthropic/claude-code-plugin-sentry
```

Para entornos que requieren colaboracion remota o integracion con servicios cloud propios, `claude remote-control` inicia un servidor de control remoto que permite dirigir una instancia de Claude Code desde otro equipo o desde un pipeline de CI/CD. Complementariamente, `claude gateway` arranca un gateway self-hosted que actua como proxy entre Claude Code y proveedores de modelos como Amazon Bedrock, Google Cloud Vertex AI o Azure OpenAI, indispensable en entornos corporativos donde el trafico no puede salir directamente a internet.

```bash
claude remote-control --port 8443
claude gateway --provider bedrock
```

Por ultimo, `claude ultrareview [target]` ejecuta un code review no interactivo sobre un directorio, archivo o incluso sobre el diff del ultimo commit. A diferencia de una revision conversacional, `ultrareview` produce un informe estructurado con problemas de seguridad, rendimiento, mantenibilidad y estilo, todo en una sola pasada sin requerir interaccion. Es la herramienta ideal para integrar en hooks de pre-commit o en pipelines de CI donde se necesita una revision automatica antes de fusionar cambios.

```bash
claude ultrareview src/
claude ultrareview --diff
```

