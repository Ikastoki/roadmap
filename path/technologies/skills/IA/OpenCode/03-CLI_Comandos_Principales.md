# OpenCode CLI

## Uso basico

El binario `opencode` es un ejecutable autónomo que encapsula un agente de inteligencia artificial, un gestor de contexto, un orquestador de herramientas y multiples interfaces de usuario en un solo comando. Cuando se invoca sin argumentos, arranca la interfaz TUI en el directorio de trabajo actual, escaneando el arbol de archivos para construir el contexto inicial del proyecto.

```bash
opencode
```

Para apuntar a un proyecto especifico se pasa la ruta como primer argumento. El CLI cambia al directorio indicado, carga la configuracion del proyecto (opencode.json, AGENTS.md) y presenta el TUI listo para interactuar:

```bash
opencode /ruta/al/proyecto
```

En modo no interactivo, `opencode run` ejecuta un prompt directamente y devuelve la respuesta. La opcion `--file` permite adjuntar un archivo al contexto:

```bash
opencode run "refactoriza esta funcion para usar async/await"
opencode run --file src/utils.ts "analiza este archivo y sugiere mejoras"
```

Internamente el CLI establece una sesion conversacional con el proveedor de IA configurado, gestiona la ventana de contexto mediante compactacion automatica, y expone un conjunto de herramientas (lectura/escritura de archivos, ejecucion de comandos, busqueda, navegacion web) que el modelo puede invocar para cumplir la tarea.

## Flags globales

Todos los subcomandos heredan un conjunto de banderas que controlan el comportamiento base del CLI. `--help` (o `-h`) muestra la documentacion de uso, mientras que `--version` (o `-v`) imprime la version instalada:

```bash
opencode --help
opencode --version
```

La flag `--print-logs` envia los registros de depuracion a stderr, util para inspeccionar el flujo interno sin mezclar la salida estandar. Combinado con `--log-level` se puede filtrar la verbosidad: `DEBUG`, `INFO`, `WARN` o `ERROR`. En nivel DEBUG se obtiene traza completa de llamadas a herramientas, intercambio de mensajes con el proveedor y decisiones del gestor de contexto, esencial para diagnostico de comportamientos inesperados.

```bash
opencode run "diagnostica este proyecto" --print-logs --log-level DEBUG
```

`--pure` ejecuta opencode sin cargar plugins externos, aislando la ejecucion del ecosistema de complementos npm. Util para entornos controlados o para verificar si un problema es causado por un plugin de terceros.

## opencode tui

El TUI es la interfaz principal del producto. Al arrancar, `opencode` (o `opencode [directorio]`) despliega un terminal interactivo dividido en panel de chat, arbol de archivos y area de resultados. La flag `--continue` (o `-c`) reanuda la sesion mas reciente, mientras que `--session` (o `-s`) reanuda una sesion concreta por ID:

```bash
opencode --continue
opencode --session abc123
```

`--fork` bifurca la sesion al reanudarla, creando una copia en lugar de modificar la original. `--prompt` inyecta un mensaje inicial en la sesion nada mas arrancar. `--model` (o `-m`) fuerza un modelo especifico en formato `proveedor/modelo`, por ejemplo `anthropic/claude-sonnet-4-20250514`:

```bash
opencode --model openai/gpt-4o --prompt "explorame la estructura del proyecto"
```

`--agent` selecciona un agente concreto (plan, build o personalizado). `--auto` concede automaticamente todos los permisos que no hayan sido explícitamente denegados, evitando las confirmaciones interactivas. `--port` y `--hostname` configuran la interfaz de red para escuchar conexiones entrantes.

## opencode run

Ejecuta un prompt de forma no interactiva y finaliza. Es el modo ideal para scripts, integracion en pipelines de CI/CD o procesos automatizados. Por defecto la salida es texto formateado legible; con `--format json` se obtienen eventos raw como estructura de datos parseable:

```bash
opencode run "explica este codigo" --format json
```

`--share` genera un enlace publico de la sesion en opncd.ai. `--attach` conecta la ejecucion a un servidor `opencode serve` en lugar de ejecutar localmente. `--thinking` muestra los bloques de razonamiento interno del modelo, incluyendo el proceso paso a paso que sigue antes de generar la respuesta final:

```bash
opencode run "debuggea este error" --thinking --attach http://servidor:4096
```

`--variant` controla el esfuerzo de razonamiento del modelo, con valores como `normal`, `high` o `low` segun el proveedor. Util para tareas que requieren mayor profundidad analitica o, por el contrario, respuestas rapidas y ligeras.

## opencode serve

Convierte opencode en un servidor headless que expone las capacidades del agente via HTTP. Es la opcion para integrar opencode como backend de otras aplicaciones, pipelines de CI o entornos multiusuario. Por defecto escucha en `localhost:4096`:

```bash
opencode serve --port 8080 --hostname 0.0.0.0
```

`--mdns` publica el servidor en la red local mediante multicast DNS, facilitando el descubrimiento desde otros dispositivos. `--cors` configura los origenes permitidos para peticiones cross-origin. La autenticacion se habilita mediante la variable de entorno `OPENCODE_SERVER_PASSWORD`; si esta presente, todas las peticiones deben incluirla como cabecera `x-api-key` o como parametro de consulta.

```bash
OPENCODE_SERVER_PASSWORD=secreto opencode serve --mdns
```

## opencode web

Idéntico a `opencode serve` pero sirviendo una interfaz web accesible desde el navegador. Permite interactuar con el agente sin instalar nada en el cliente, util para entornos de formacion, demos o equipos que prefieren no usar el terminal. Acepta las mismas opciones que serve.

```bash
opencode web --port 3000 --hostname 0.0.0.0
```

## opencode attach

Conecta el TUI local a un servidor remoto, delegando toda la computacion al nodo servidor mientras la interfaz se ejecuta en el cliente. La direccion del servidor se pasa como argumento:

```bash
opencode attach http://192.168.1.100:4096
```

Si el servidor requiere autenticacion, se proporcionan `--password` y opcionalmente `--username`. El TUI remoto funciona con la misma experiencia que el local: historial, autocompletado, comandos slash y todas las herramientas, pero ejecutadas contra el proyecto alojado en el servidor.

```bash
opencode attach https://opencode.miempresa.com --password token-seguro
```

## opencode session

Gestiona las sesiones almacenadas localmente. `session list` enumera las sesiones disponibles con metadatos (modelo, fecha, proyecto, resumen). `--max-count` limita el numero de resultados y `--format` permite salida en json para procesamiento automatico:

```bash
opencode session list --max-count 10 --format json
```

`session delete <id>` elimina una sesion concreta y todos sus datos asociados. No hay confirmacion interactiva, por lo que conviene verificar primero el identificador con `list`.

```bash
opencode session delete abc123
```

## opencode stats

Proporciona estadisticas de uso: tokens consumidos, costos acumulados por proveedor y numero de llamadas. Es la herramienta para monitorizar el gasto y optimizar la seleccion de modelos. Los filtros permiten acotar el periodo y la granularidad:

```bash
opencode stats --days 30 --models --tools --project
```

`--days` limita el rango temporal. `--models` desglosa por modelo. `--tools` muestra el uso de cada herramienta. `--project` agrupa por proyecto. Sin flags adicionales muestra un resumen global del periodo indicado.

## opencode export / import

Permite serializar y transportar sesiones entre entornos. `export <id>` vuelca la sesion completa a JSON, incluyendo historial de mensajes, archivos modificados y metadatos. `--sanitize` redacta valores sensibles como claves API o tokens antes de exportar:

```bash
opencode export sesion-001 --sanitize > respaldo.json
```

`import` restaura una sesion desde un archivo local o una URL. Util para compartir contextos de trabajo entre desarrolladores o migrar sesiones de un entorno a otro sin perder el historial:

```bash
opencode import https://opncd.ai/share/abc123
opencode import respaldo.json
```

> La importacion de sesiones de fuentes no verificadas debe hacerse con precaucion, ya que el JSON puede contener instrucciones o configuraciones que alteren el comportamiento del agente.

