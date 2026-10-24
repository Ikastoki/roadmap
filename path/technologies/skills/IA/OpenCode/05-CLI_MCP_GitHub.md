# OpenCode CLI

La integracion con servicios externos y la capacidad de operar dentro del ecosistema GitHub transforman a OpenCode de un asistente local de codigo a un agente que participa activamente en los flujos de trabajo del equipo. Los comandos de gestion MCP (Model Context Protocol) permiten conectar el agente con fuentes de datos y herramientas externas, mientras que los subcomandos `github` y `pr` habilitan la interaccion automatizada con pull requests, code reviews y pipelines de integracion continua. Adicionalmente, el comando `db` proporciona visibilidad directa sobre el almacenamiento local que OpenCode utiliza para sesiones, configuracion y cache, cerrando el circulo entre la operacion del agente y los datos que genera.

## opencode mcp

El comando `opencode mcp` gestiona los servidores MCP que extienden las capacidades del agente. MCP es un protocolo abierto que define como los modelos de lenguaje pueden interactuar con herramientas externas: un servidor MCP expone recursos (datos) y herramientas (funciones) que OpenCode puede invocar durante una conversacion como si fueran herramientas nativas. La diferencia fundamental con una integracion via API convencional es que el agente descubre las capacidades del servidor en tiempo de ejecucion, sin necesidad de configurar puntos de conexion especificos en el prompt.

Los servidores MCP pueden ser locales o remotos. Un servidor local se ejecuta como un proceso hijo de OpenCode, tipicamente lanzado mediante un comando como `npx` o `python`. Un servidor remoto se comunica via HTTP y puede estar alojado en cualquier infraestructura accesible por red. Los servidores locales son adecuados para herramientas especificas del proyecto (linters, generadores de documentacion, analizadores estaticos), mientras que los remotos permiten integrar servicios compartidos como sistemas de monitorizacion, bases de datos centralizadas o APIs de terceros.

```bash
# Anadir un servidor MCP interactivamente
opencode mcp add

# Listar servidores configurados y su estado
opencode mcp list
opencode mcp ls

# Depurar una conexion MCP especifica
opencode mcp debug sentry

# Autenticar un servidor MCP remoto via OAuth
opencode mcp auth

# Eliminar credenciales OAuth de un servidor
opencode mcp logout
```

Cuando un servidor MCP esta activo, sus herramientas se fusionan con el conjunto de herramientas integradas de OpenCode. El agente puede decidir automaticamente que herramienta usar segun la tarea: si el problema requiere buscar un error en Sentry, invocara la herramienta que expone el servidor MCP de Sentry; si necesita leer un archivo local, usara la herramienta `read` nativa. La configuracion de MCP se almacena en `opencode.json` bajo la clave `mcpServers`, donde se definen el tipo (`local` o `remote`), el comando de arranque o la URL, las variables de entorno y los timeouts especificos.

> Los permisos de las herramientas MCP se gestionan independientemente de las herramientas integradas. Es posible permitir, preguntar o denegar el uso de cada herramienta MCP por agente mediante los patrones de permisos en la configuracion.

## opencode github

El comando `opencode github` integra a OpenCode con la plataforma GitHub, permitiendo que el agente participe en flujos de trabajo automatizados dentro de repositorios. La integracion se materializa mediante la instalacion de un agente GitHub Actions que, al ejecutarse, descarga el codigo del repositorio, lanza OpenCode y le proporciona el contexto necesario para realizar tareas de revision de codigo, auditoria de seguridad o aplicacion automatica de cambios.

```bash
# Instalar el agente GitHub en el repositorio actual
opencode github install

# Ejecutar el agente GitHub (tipicamente dentro de Actions)
opencode github run --event pull_request --token ghp_xxxx
```

La instruccion `github install` genera y configura el workflow de GitHub Actions necesario. El workflow incluye pasos para checkout del codigo, configuracion del entorno, autenticacion y ejecucion de `opencode github run`. El flag `--event` indica el tipo de evento que desencadeno la ejecucion (por ejemplo, `pull_request`, `push`, `issue_comment`) y `--token` proporciona el token de acceso necesario para que OpenCode interactue con la API de GitHub (comentar en PRs, actualizar estados, crear commits).

La potencia de esta integracion reside en que OpenCode opera con las mismas capacidades que en local, pero dentro de un entorno de CI. Puede analizar el diff de un PR, identificar problemas de estilo, vulnerabilidades de seguridad o inconsistencias arquitectonicas, y publicar sus hallazgos como comentarios en el PR. Tambien puede proponer cambios concretos y, si la configuracion de permisos lo permite, aplicarlos directamente mediante commits.

## opencode pr

El comando `opencode pr` proporciona un flujo de trabajo simplificado para trabajar con pull requests de GitHub. Dado un numero de PR, OpenCode se conecta a la API de GitHub, obtiene los metadatos del pull request, descarga la rama asociada y la fusiona con la base, y luego ejecuta el agente sobre el codigo resultante. Esto permite que el usuario solicite revisiones, cambios o analisis directamente desde la terminal sin cambiar de contexto.

```bash
# Fetch, checkout y ejecucion de OpenCode sobre un PR
opencode pr 42
```

Durante la ejecucion, el agente tiene acceso al diff del PR, al historial de commits de la rama y a todo el arbol del proyecto en el estado resultante de la fusion. Puede realizar revisiones de codigo, ejecutar el conjunto de tests, verificar que se cumplen las politicas del proyecto definidas en `AGENTS.md` o incluso modificar los archivos del PR para corregir problemas detectados. Al finalizar, OpenCode publica un resumen en el hilo del PR con los hallazgos y las acciones realizadas.

El flujo de `opencode pr` esta disenado para integrarse con `opencode github`: mientras que `github run` se ejecuta automaticamente en CI ante eventos, `pr` se invoca manualmente desde la terminal de un desarrollador, proporcionando un punto de control previo a la融合 definitiva del PR.

> Para que `opencode pr` funcione correctamente, el repositorio debe tener configurada la autenticacion con GitHub. OpenCode utiliza el token proporcionado en `github install` o la variable de entorno `GITHUB_TOKEN`.

## opencode db

El comando `opencode db` expone el almacenamiento interno de OpenCode para inspeccion y consulta directa. OpenCode mantiene una base de datos local donde persiste el estado de las sesiones, la configuracion resuelta, la cache de modelos y los metadatos de uso. Esta base de datos reside en el directorio de datos de la aplicacion (`~/.local/share/opencode/` en Linux, rutas equivalentes en macOS y Windows) y utiliza un formato estructurado que puede consultarse desde la linea de comandos.

```bash
# Ejecutar una consulta SQL sobre la base de datos
opencode db "SELECT id, created_at FROM sessions ORDER BY created_at DESC LIMIT 10"

# Mostrar la ruta absoluta del archivo de base de datos
opencode db path

# Exportar resultados en formato JSON
opencode db "SELECT * FROM sessions" --format json

# Exportar resultados en formato TSV
opencode db "SELECT * FROM sessions" --format tsv
```

Los datos almacenados incluyen el historial de mensajes de cada sesion, la configuracion activa (con todas las variables resueltas), el registro de llamadas a modelos con sus costes asociados, la cache de respuestas para evitar recomputaciones, y los metadatos de las herramientas utilizadas. Esta informacion es valiosa para auditar el comportamiento del agente, depurar problemas de configuracion, analizar patrones de uso o simplemente liberar espacio eliminando sesiones antiguas.

La opcion `db path` es especialmente util en entornos donde la ubicacion de los datos puede variar debido a variables de entorno como `XDG_DATA_HOME` o `OPENCODE_DATA_DIR`. Conocer la ruta exacta permite acceder directamente al archivo con otras herramientas o realizar copias de seguridad del estado completo de OpenCode.

