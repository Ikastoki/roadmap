# OpenCode CLI

## opencode auth

El sistema de autenticacion de OpenCode gestiona las credenciales de todos los proveedores de lenguaje desde un unico punto de entrada. El comando `opencode auth login` inicia el flujo de configuracion de una API key para un proveedor especifico. La opcion `--provider` (o `-p`) identifica el proveedor mediante su ID estandar (`anthropic`, `openai`, `google`, `groq`, etc.), mientras que `--method` (o `-m`) selecciona el mecanismo de autenticacion: `key` para una API key directa, `oauth` para flujos OAuth, o `env` para leer la clave desde una variable de entorno ya definida. Sin argumentos, el comando ofrece un asistente interactivo que guia al usuario en la seleccion del proveedor y la introduccion de la credencial.

```bash
opencode auth login --provider anthropic --method key
opencode auth login -p openai -m env
opencode auth login
```

Las credenciales no se almacenan en texto plano dentro de la configuracion del proyecto. OpenCode las guarda en el keychain del sistema operativo cuando esta disponible (Keychain en macOS, libsecret en Linux, Credential Manager en Windows), y solo como fallback las escribe en el archivo de configuracion `~/.config/opencode/config.json`. El comando `opencode auth list` (o su alias `ls`) muestra todos los proveedores autenticados, indicando el metodo usado y si la sesion esta activa. Para revocar una credencial, `opencode auth logout` cierra la sesion del proveedor eliminando la clave almacenada. OpenCode soporta multiples proveedores de forma simultanea: cuando se ejecuta una tarea, el sistema intenta con el proveedor primario definido en la configuracion y, si falla (por limite de tasa, caida del servicio o agotamiento de credito), realiza un fallback automatico al siguiente proveedor disponible en orden de prioridad. Este comportamiento permite construir pipelines resilientes donde la caida de un proveedor no interrumpe el flujo de trabajo.

```bash
opencode auth list
opencode auth logout --provider openai
```

> El orden de fallback entre proveedores se define en `opencode.json` mediante la propiedad `provider`. Si no se especifica, OpenCode usa el orden por defecto segun disponibilidad: Anthropic, OpenAI, Google, y el resto en orden alfabetico.

## opencode models

OpenCode mantiene un catalogo local de modelos obtenido mediante consultas directas a las APIs de cada proveedor autenticado. El comando `opencode models` lista todos los modelos disponibles, mostrando su nombre completo en el formato `provider/model` (por ejemplo, `anthropic/claude-sonnet-4-20250514`, `openai/gpt-4o`, `google/gemini-2.5-pro`). Es posible filtrar por proveedor anadiendo el ID como argumento posicional: `opencode models anthropic` muestra unicamente los modelos de Anthropic. La primera vez que se ejecuta, OpenCode consulta cada API y construye el catalogo; las respuestas se almacenan en cache local para evitar llamadas repetidas. El flag `--refresh` forza una consulta nueva a todos los proveedores, ignorando la cache, util cuando un proveedor lanza nuevos modelos o actualiza su catalogo de precios.

```bash
opencode models
opencode models openai
opencode models --refresh
opencode models --verbose
```

La convencion `provider/model` es el identificador universal que OpenCode utiliza en todos los comandos que aceptan un modelo. Cuando se pasa `--model anthropic/claude-sonnet-4-20250514`, el sistema parsea el prefijo para determinar que proveedor debe usar y con que credencial autenticarse. El flag `--verbose` anade metadatos adicionales a cada modelo: capacidad de vision, soporte para herramientas, ventana de contexto maxima en tokens, y precios por millon de tokens de entrada y salida. OpenCode utiliza esta informacion de costos para el seguimiento de gastos integrado en `opencode stats`, permitiendo al usuario monitorizar el consumo de cada modelo y proveedor a lo largo del tiempo. El catalogo incluye tanto los modelos propietarios de cada proveedor como modelos open source servidos a traves de plataformas como Groq, Together AI o Fireworks AI.

```bash
opencode models --verbose | grep "claude"
opencode models anthropic --refresh --verbose
```

> La cache de modelos se almacena en `~/.local/share/opencode/models.json`. OpenCode respeta los headers `Cache-Control` de las APIs, pero `--refresh` invalida la cache independientemente de su estado. El catalogo se actualiza automaticamente cada 24 horas como parte del comportamiento por defecto.

## opencode agent

OpenCode permite definir agentes personalizados que heredan y extienden el comportamiento de los agentes integrados. El comando `opencode agent create` inicia un asistente interactivo que solicita el nombre, la descripcion, el modo (`primary`, `subagent` o `all`), el modelo por defecto y los permisos de las herramientas. Para entornos automatizados o scripting, todas estas opciones estan disponibles como flags: `--path` para la ruta del archivo del agente, `--description` para el proposito, `--mode` para el tipo de agente, `--permissions` para el nivel de acceso a herramientas, y `--model` para el modelo por defecto. Cada agente se materializa en un archivo Markdown con frontmatter YAML que OpenCode procesa al arrancar.

```bash
opencode agent create
opencode agent create --path .opencode/agents/revisor.md --description "Code review specialist" --mode subagent --model anthropic/claude-sonnet-4-20250514 --permissions read
opencode agent list
```

La definicion de un agente personalizado sigue la misma estructura que los agentes integrados del sistema. El archivo comienza con un bloque YAML delimitado por `---` que contiene la configuracion (modelo, temperatura, top_p, permisos, herramientas habilitadas), seguido de un cuerpo en Markdown que constituye el prompt del sistema. OpenCode carga estos agentes desde dos ubicaciones: `~/.config/opencode/agents/` para agentes globales disponibles en cualquier proyecto, y `.opencode/agents/` para agentes locales vinculados a un proyecto especifico. La precedencia es local sobre global, permitiendo sobrescribir agentes del mismo nombre. El comando `opencode agent list` enumera todos los agentes disponibles, indicando su origen (integrado, global o local) y su modo, para que el usuario pueda seleccionarlos con `--agent` o mediante la paleta de comandos del TUI.

```bash
opencode agent list
opencode agent list --verbose
```

Los agentes personalizados heredan implicitamente las herramientas del sistema base, pero el sistema de permisos permite un control granular sobre que herramientas puede ejecutar cada agente. La propiedad `permission` acepta valores como `read` (solo herramientas de lectura), `write` (lectura y escritura), `full` (todas las herramientas incluyendo bash), o un objeto JSON con permisos especificos por herramienta. Este modelo de seguridad permite disenar agentes de proposito limitado, como un revisor de codigo que solo puede leer archivos y ejecutar comandos de linting, o un auditor de seguridad que tiene acceso restringido a herramientas de red pero no puede modificar el codigo fuente. La integracion con los agentes integrados se produce mediante la seleccion primaria en el TUI: al presionar `Tab` se cicla entre todos los agentes primarios disponibles, tanto los integrados como los personalizados, permitiendo una transicion fluida entre modos de trabajo.

```bash
# Ejemplo de agente personalizado: .opencode/agents/revisor.md
# ---
# description: "Code review specialist"
# mode: subagent
# model: anthropic/claude-sonnet-4-20250514
# permission:
#   read: allow
#   bash: ask
#   write: deny
#   edit: deny
# ---
```

