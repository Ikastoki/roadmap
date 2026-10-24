# OpenCode CLI

El sistema de configuracion de OpenCode sigue una arquitectura por capas que recuerda a las pilas de configuracion de red: cada nivel puede anular al anterior y la resolucion final es la union de todos ellos, con la maxima precedencia ganando en caso de conflicto. Este diseno permite que la herramienta se comporte de forma distinta segun el contexto: un proyecto puede declarar sus propios ajustes sin afectar a la configuracion global, los entornos de CI pueden inyectar sobreescrituras mediante variables de entorno y los despliegues empresariales pueden imponer politicas a traves de configuracion gestionada. El archivo de configuracion, denominado `opencode.json` o `opencode.jsonc`, es la pieza central de este sistema y el primer lugar donde OpenCode busca las directrices de funcionamiento al iniciar una sesion.

## opencode.json / opencode.jsonc

OpenCode acepta tanto JSON puro como JSON con comentarios. El soporte de comentarios, habilitado mediante la extension `.jsonc`, permite anotar cada opcion con su proposito sin recurrir a documentacion externa. El esquema completo esta publicado en `https://opencode.ai/config.json` y puede utilizarse para validacion en editores y pipelines de CI/CD. Muchos editores de codigo, al detectar la presencia de un archivo `opencode.json` en la raiz del proyecto, ofrecen autocompletado y validacion en tiempo real si se configura el enlace al schema.

La precedencia de configuracion se resuelve en siete niveles. En el nivel mas bajo se encuentran los valores por defecto que OpenCode incorpora: un conjunto de opciones sensatas que garantizan que la herramienta funcione sin necesidad de configuracion previa. El segundo nivel es la configuracion global, almacenada en `~/.config/opencode/opencode.json`, que aplica a todas las sesiones del usuario independientemente del directorio de trabajo. El tercer nivel es la configuracion del proyecto, ubicada en `opencode.json` o `opencode.jsonc` en la raiz del repositorio; este es el archivo que normalmente se versiona y se comparte con el equipo. El cuarto nivel son las banderas de linea de comandos, que permiten sobreescribir opciones en una invocacion concreta sin modificar archivos. El quinto nivel corresponde a las variables de entorno, con nombres como `OPENCODE_MODEL` o `OPENCODE_PERMISSION`. El sexto nivel incluye las sobreescrituras definidas en prompts o configuraciones de agentes especificos, como las que aparecen en `AGENTS.md` o en la definicion de un agente personalizado. El septimo y ultimo nivel es la configuracion gestionada por MDM (Mobile Device Management) o politicas corporativas, que prevalece sobre cualquier otra.

La resolucion de la configuracion funciona mediante una fusion profunda de objetos: los valores simples se sobrescriben con la fuente de mayor precedencia, mientras que los objetos complejos como `provider` o `agent` se fusionan recursivamente. Esto significa que la configuracion global puede anadir un proveedor y la configuracion del proyecto puede anadir otro, y ambos estaran disponibles al mismo tiempo. Sin embargo, si ambos niveles definen el mismo proveedor con la misma clave, gana el de mayor precedencia.

```json
{
  "model": "anthropic/claude-sonnet-4-20250620",
  "small_model": "gpt-4o-mini",
  "provider": {
    "anthropic": {
      "apiKey": "{env:ANTHROPIC_API_KEY}"
    },
    "openai": {
      "apiKey": "{env:OPENAI_API_KEY}",
      "baseUrl": "https://api.openai.com/v1"
    }
  },
  "shell": {
    "bash": {
      "command": "bash",
      "args": ["--login"]
    }
  },
  "permission": {
    "bash": "ask",
    "write": "allow",
    "glob": "allow"
  }
}
```

## Opciones principales

La opcion `model` define el modelo por defecto para todas las sesiones, expresado en el formato `provider/model`. Este modelo se utiliza tanto en el modo TUI como en el modo CLI, a menos que se sobreescriba mediante la bandera `--model` o en la configuracion de un agente especifico. La opcion `small_model` define un modelo secundario, generalmente mas rapido y economico, que OpenCode utiliza para tareas ligeras como resumir contexto, generar titulos de sesion o analisis preliminares de archivos. Separar el modelo principal del modelo pequeno permite optimizar costes sin sacrificar calidad en las tareas que realmente lo requieren.

El bloque `provider` contiene la configuracion de cada proveedor de modelos, incluyendo claves de API, URL base y parametros especificos como `maxTokens` o `temperature` por defecto. OpenCode soporta multiples proveedores simultaneamente y selecciona el modelo segun la configuracion de `model` o `small_model`. La opcion `shell` determina que shell se utiliza para ejecutar comandos interactivos; por defecto OpenCode detecta el shell del sistema, pero puede forzarse a bash, zsh, fish o PowerShell segun las necesidades del proyecto. La opcion `tools` permite habilitar o deshabilitar herramientas integradas de forma granular, escribiendo `true` o `false` junto al nombre de cada herramienta.

El sistema de permisos se controla mediante `permission`, donde cada herramienta puede asignarse a `"allow"`, `"ask"` o `"deny"`. El valor `"allow"` ejecuta la herramienta sin confirmacion, `"ask""` muestra una confirmacion antes de cada uso y `"deny"` bloquea la herramienta por completo. Para la herramienta `bash`, los permisos pueden refinarse con patrones de comando: `"bash": {"git *": "allow", "*": "ask"}` permite los comandos git sin preguntar pero solicita confirmacion para cualquier otro. La opcion `agent` contiene la configuracion de agentes personalizados, donde cada agente puede definir su propio modelo, conjunto de herramientas, permisos y prompt de sistema.

```json
{
  "tools": {
    "write": true,
    "edit": true,
    "bash": true,
    "fetch": true,
    "websearch": true
  },
  "permission": {
    "bash": {
      "git *": "allow",
      "npm *": "allow",
      "*": "ask"
    },
    "write": "allow",
    "glob": "allow"
  },
  "disabled_providers": ["groq", "together"],
  "enabled_providers": null,
  "share": "manual",
  "autoupdate": true,
  "snapshot": {
    "enabled": true,
    "maxSnapshots": 50
  },
  "compaction": {
    "strategy": "auto",
    "tokenLimit": 64000
  }
}
```

Las opciones `mcp`, `command`, `lsp` y `formatter` extienden las capacidades de OpenCode mas alla de las herramientas integradas. `mcp` define servidores del Model Context Protocol, tanto locales (ejecutados como procesos hijo) como remotos (conexiones HTTP). `command` permite definir alias de comandos personalizados que pueden invocarse desde el prompt con `!nombre`. `lsp` habilita la integracion con servidores de lenguaje para diagnostico en tiempo real, soportando TypeScript, Python, Java, Rust y otros lenguajes mediante sus respectivos LSP. `formatter` asigna formateadores de codigo por extension de archivo. `plugin` permite instalar plugins desde npm que extienden la funcionalidad de OpenCode con herramientas personalizadas. `instructions` acepta una lista de rutas de archivos o URLs que contienen instrucciones adicionales, con soporte para glob patterns, de modo que `"packages/*/AGENTS.md"` carga automaticamente las reglas de cada subpaquete.

Los modificadores `disabled_providers` y `enabled_providers` controlan que proveedores estan disponibles. Si se usa `enabled_providers`, funciona como lista de permitidos: solo los proveedores listados estaran disponibles, y todos los demas se ignoran. Si se usa `disabled_providers`, funciona como lista de denegados: los proveedores listados se excluyen y el resto permanecen accesibles. La opcion `share` determina el comportamiento de la funcionalidad para compartir sesiones mediante enlaces `opncd.ai`: `"manual"` requiere que el usuario ejecute `/share` explicitamente, `"auto"` comparte automaticamente cada sesion y `"disabled"` desactiva la funcionalidad por completo.

El control de recursos se gestiona mediante `snapshot`, `compaction` y `watcher`. `snapshot` activa el sistema de instantaneas que permite deshacer y rehacer cambios a nivel de operacion, con un limite configurable de instantaneas almacenadas. `compaction` controla la compactacion del contexto de la sesion: cuando el numero de tokens supera el limite definido, OpenCode resume automaticamente los intercambios mas antiguos para mantener la sesion dentro de la ventana de contexto del modelo. `watcher` define patrones de ignorado para el observador de archivos, evitando que modificaciones en directorios como `node_modules` o `.git` disparen eventos innecesarios. `experimental` agrupa opciones en fase de prueba que pueden cambiar o desaparecer en versiones futuras.

## Variables en config

OpenCode admite la interpolacion de variables dentro de los valores de configuracion mediante dos sintaxis. `{env:VARIABLE}` se resuelve al valor de la variable de entorno `VARIABLE` en el momento en que OpenCode lee la configuracion. `{file:path}` se resuelve al contenido del archivo en la ruta indicada. Esta capacidad es fundamental para mantener la seguridad de las credenciales: las claves de API nunca deben escribirse directamente en el archivo de configuracion, ya que este suele versionarse en el repositorio.

```json
{
  "provider": {
    "anthropic": {
      "apiKey": "{env:ANTHROPIC_API_KEY}"
    },
    "openai": {
      "apiKey": "{env:OPENAI_API_KEY}"
    },
    "github": {
      "apiKey": "{file:~/.config/github-token}"
    }
  }
}
```

La interpolacion tambien puede combinarse con valores literales, aunque en la practica se recomienda mantener las variables de entorno como unico mecanismo para secretos, y reservar `{file:path}` para certificados, tokens largos o cualquier valor que por su extension sea mas comodo mantener en un archivo separado. OpenCode evalua estas variables en el momento de la carga de configuracion y, si una variable de entorno no esta definida o un archivo no existe, la resolucion falla con un mensaje de error claro que indica exactamente que variable o archivo falta.

> Incluir claves de API directamente en `opencode.json` es una practica insegura. Ademas de exponer las credenciales en el sistema de control de versiones, cualquier persona con acceso al repositorio obtiene las claves. El uso de `{env:VARIABLE}` mantiene los secretos fuera del codigo y delega su gestion al administrador del sistema, al gestor de secretos del entorno de CI o al archivo `.env` del desarrollador.

```json
{
  "provider": {
    "azure": {
      "apiKey": "{env:AZURE_OPENAI_KEY}",
      "baseUrl": "{env:AZURE_OPENAI_ENDPOINT}",
      "deploymentName": "gpt-4"
    }
  }
}
```

