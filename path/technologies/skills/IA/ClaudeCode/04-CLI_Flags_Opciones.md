# Claude Code CLI

## Flags de sesion

La linea de comandos de Claude Code se compone de un conjunto de banderas que modifican el comportamiento de la sesion desde el momento de la invocacion. El flag `--model` permite seleccionar el modelo subyacente entre las variantes disponibles: `sonnet` (predeterminado), `opus` y `haiku`. Cada uno ofrece una relacion distinta entre capacidad de razonamiento, velocidad y costo. El flag `--agent` acepta un identificador de agente personalizado, lo que permite cargar una configuracion especifica de instrucciones, herramientas y permisos definida previamente en el archivo de configuracion del proyecto.

El sistema de permisos se controla mediante `--permission-mode`, que acepta tres valores: `plan` (modo solo lectura, equivalente al agente Plan), `auto` (clasificador automatico que decide si ejecutar o preguntar) y `bypassPermissions` (salta todas las confirmaciones). Para refinar este comportamiento existen `--allowedTools`, que lista herramientas que se ejecutan sin solicitar permiso, y `--disallowedTools`, que las deniega directamente. La integracion con el sistema de archivos se extiende mediante `--add-dir`, que anade directorios adicionales fuera del proyecto al alcance del agente.

> El flag `--dangerously-skip-permissions` es un alias directo de `--permission-mode bypassPermissions` y debe usarse unicamente en entornos controlados como CI.

Para inyectar instrucciones adicionales al sistema sin modificar los archivos de configuracion permanente, se emplean `--append-system-prompt` con texto literal y `--append-system-prompt-file` que carga el contenido de un archivo de reglas. Ambos se concatenan al final del system prompt y son utiles para contextos temporales como code reviews o auditorias.

```bash
claude --model opus --permission-mode plan --add-dir /etc/nginx
claude --agent code-reviewer --allowedTools read,grep,glob
claude --append-system-prompt "Prioriza la seguridad por encima del rendimiento"
claude --append-system-prompt-file ./regles_seguretat.md
```

## Flags tecnicos

La depuracion se activa con `--debug` y admite filtros por categoria separados por comas, como `"api,mcp"`, para limitar la salida a los subsistemas de interes. Cuando se necesita persistir los registros, `--debug-file <path>` redirige toda la traza a un archivo independientemente de la salida estandar. El flag `--bare` inicia Claude Code en modo minimo, sin autodetectar hooks, skills, plugins, servidores MCP ni el archivo `CLAUDE.md`, lo que resulta util para entornos de pruebas o escenarios donde se requiere un control deterministico del entorno.

La accesibilidad se aborda con `--ax-screen-reader`, que produce una salida sin bordes, animaciones ni elementos visuales que interfieran con los lectores de pantalla. La integracion con el navegador se habilita mediante `--chrome`, que permite a Claude Code interactuar con una instancia de Chrome para tareas de depuracion web. El flag `--cloud` crea una sesion web vinculada a `claude.ai`, mientras que `--betas` inyecta cabeceras beta en las llamadas a la API para acceder a funcionalidades en fase de prueba.

```bash
claude --debug "api,mcp" --debug-file /tmp/claude_debug.log
claude --bare --model haiku
claude --chrome --cloud
claude --betas "computer-use-2025-01-24"
```

## Flags de gestion

Claude Code incluye subcomandos de administracion del propio agente que operan fuera del contexto de una sesion. `claude update` verifica e instala la version mas reciente. `claude doctor` ejecuta un diagnostico completo del entorno: verifica la conexion con la API, el estado del sistema de archivos, los permisos y la configuracion activa, y reporta cualquier anomalia detectada. Para entornos de integracion continua, `claude setup-token` configura un token OAuth que permite la autenticacion no interactiva.

El modo servidor se despliega con `claude remote-control`, que inicia un servidor remoto para control externo, y `claude gateway`, que actua como gateway empresarial con capacidades de enrutamiento, autenticacion y auditoria centralizada. `claude project purge` elimina los datos locales de un proyecto especifico, incluyendo sesiones, cache y configuracion asociada. El supervisor del agente se gestiona mediante `claude daemon status` para consultar su estado y `claude daemon stop` para detenerlo.

```bash
claude update
claude doctor
claude setup-token --token mi-token-oauth
claude project purge
claude daemon status
claude gateway --port 8080
```

> El subcomando `claude doctor` es la primera herramienta de diagnostico ante problemas de conexion, permisos o configuracion. Su salida incluye recomendaciones accionables para cada problema detectado.

