# Claude Code CLI

## settings.json

Claude Code centraliza su configuracion en el archivo `settings.json`, un documento JSON que define desde el modelo de lenguaje que impulsa las respuestas hasta los directorios que el agente puede modificar. Este archivo no es un mero listado de preferencias: constituye la superficie de control del comportamiento del agente, determinando que herramientas puede ejecutar, bajo que restricciones opera y como se presenta la informacion al usuario. La gestion de esta configuracion opera por capas con una jerarquia de precedencia clara, permitiendo que cada nivel del ecosistema -- el desarrollador individual, el equipo del proyecto y la organizacion -- imponga sus politicas sin conflicto.

```json
{
  "model": "claude-sonnet-4-20250624",
  "theme": "catppuccin-mocha",
  "outputStyle": "markdown",
  "allowedTools": ["Read", "Edit", "Bash", "Glob", "Grep"],
  "disallowedTools": ["WebFetch"],
  "permissions": {
    "additionalDirectories": ["/home/usuario/shared-libs", "/var/data"]
  },
  "autoMode": {
    "enabled": true,
    "classifierThreshold": 0.85,
    "allowedCategories": ["file_read", "code_search", "terminal_output"]
  }
}
```

El nivel de proyecto -- `./.claude/settings.json` -- es el de mayor prioridad y suele incluirse en el repositorio para garantizar que todo miembro del equipo opere con la misma configuracion base. Le sigue el nivel de usuario en `~/.claude/settings.json`, que permite a cada desarrollador sobrescribir preferencias personales sin afectar al equipo: cambiar el tema, ajustar atajos de teclado o anadir directorios locales de confianza. En el escalon inferior se encuentran los managed settings, gestionados centralmente mediante MDM (Mobile Device Management) empresarial, que imponen restricciones de seguridad y cumplimiento normativo que ni el usuario ni el proyecto pueden eludir. Esta estratificacion sigue el principio de privilegio minimo: la configuracion mas especifica prevalece, pero las politicas corporativas actuan como cota inferior infranqueable.

Durante una sesion interactiva, los comandos `/config` o `/settings` permiten inspeccionar y modificar parametros en tiempo real sin editar manualmente el archivo JSON. La sintaxis es `clave=valor` y pueden concatenarse multiples ajustes en una sola linea:

```bash
/config model=claude-sonnet-4-20250624 theme=catppuccin-mocha effort=high
```

## Opciones principales

La opcion `model` define el modelo de lenguaje que procesa las solicitudes. Claude Code soporta toda la familia Claude 4: desde `claude-sonnet-4-20250624` como equilibrio entre velocidad y capacidad hasta `claude-opus-4-20260514` para tareas que requieren razonamiento profundo. El parametro `agent` permite especificar un agente personalizado definido en el proyecto, desacoplando la configuracion del modelo base de la logica de comportamiento del agente. `allowedTools` y `disallowedTools` ofrecen control granular sobre el arsenal operativo del agente: el primero lista explicitamente las herramientas que puede utilizar (Read, Edit, Bash, Glob, Grep, WebFetch, entre otras), mientras que el segundo bloquea herramientas especificas incluso si estuvieran habilitadas por configuracion de nivel superior. Esta dualidad permite construir perfiles de seguridad finos, donde un equipo de backend puede restringir herramientas de red mientras otro de frontend las necesita.

```json
{
  "advisorModel": "claude-sonnet-4-20250624",
  "autoMode": {
    "enabled": false,
    "classifierThreshold": 0.92,
    "allowedCategories": ["file_read", "code_search"],
    "blockedCategories": ["file_write", "command_execution"]
  },
  "hooks": {
    "PreToolUse": {
      "command": "node scripts/validate-tool-use.js",
      "timeout": 5000
    },
    "PostToolUse": {
      "command": "node scripts/log-tool-use.js"
    }
  }
}
```

El campo `permissions.additionalDirectories` extiende el alcance del agente mas alla del directorio de trabajo, declarando rutas adicionales que Claude Code puede leer o modificar. Sin esta declaracion explicita, el agente rechazara operaciones fuera del arbol del proyecto, una salvaguarda contra accesos no deseados a datos sensibles. `advisorModel` configura un modelo separado para la herramienta advisor, que actua como segundo par de ojos en revisiones de codigo y analisis de seguridad; al delegar esta funcion a un modelo distinto se evita el sesgo de confirmacion y se obtienen perspectivas complementarias.

`autoMode` configura el clasificador automatico que decide, ante una solicitud del usuario, si la operacion puede ejecutarse directamente o requiere confirmacion explicita. El umbral `classifierThreshold` (valor entre 0 y 1) determina la agresividad del clasificador: valores cercanos a 1 reducen los falsos positivos pero pueden dejar pasar operaciones de riesgo, mientras que valores bajos incrementan la supervision a costa de friccion. Las categorias `allowedCategories` y `blockedCategories` definen que tipo de operaciones entran en el escrutinio automatico, desde lectura de archivos hasta ejecucion de comandos en el terminal.

```json
{
  "theme": "catppuccin-mocha",
  "outputStyle": "markdown",
  "keybindings": {
    "ctrl+k": "comando.clear",
    "ctrl+l": "comando.compact",
    "ctrl+shift+p": "menu.comandos",
    "alt+enter": "accion.ejecutar"
  },
  "mcpServers": {
    "github-actions": {
      "command": "npx",
      "args": ["@anthropic/mcp-github-actions", "--token", "${GITHUB_TOKEN}"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "database-schema": {
      "command": "node",
      "args": ["server-mcp.js"],
      "transport": "stdio"
    }
  }
}
```

`theme` y `outputStyle` gobiernan la experiencia visual en el terminal. Los temas disponibles incluyen paletas como `catppuccin-mocha`, `dracula`, `nord` o `solarized-dark`, que se aplican a la barra de estado, los paneles laterales y los resaltados de sintaxis. `outputStyle` determina el formato de las respuestas del agente: `markdown` produce salida formateada con bloques de codigo y tablas, `plain` genera texto sin procesar para terminales minimalistas y `json` estructura la salida para consumo programatico. `keybindings` permite redefinir el mapa completo de atajos de teclado, asociando combinaciones como `ctrl+k` a comandos slash (`comando.clear`, `comando.compact`) o a acciones directas como `menu.comandos` o `accion.ejecutar`. Esta capa de personalizacion es esencial para usuarios que pasan horas en el terminal y necesitan optimizar cada movimiento.

Los `hooks` constituyen el sistema de extension comportamental de Claude Code. `PreToolUse` ejecuta un script antes de que el agente invoque cualquier herramienta, permitiendo validar la operacion, registrar intentos o incluso denegar la ejecucion segun logica personalizada. `PostToolUse` se dispara tras completarse la operacion, ideal para auditoria, notificaciones o pipelines de integracion continua. Cada hook recibe un payload JSON con el contexto de la invocacion (herramienta, argumentos, resultado, duracion) y debe completarse dentro del timeout configurado. `mcpServers` conecta Claude Code con servidores MCP (Model Context Protocol), exponiendo herramientas externas como si fueran nativas: desde integraciones con GitHub Actions hasta acceso a esquemas de bases de datos o APIs internas de la organizacion. Cada entrada define el comando de arranque, los argumentos y las variables de entorno necesarias, soportando interpolacion de variables segun el formato `${VARIABLE}`.

```json
{
  "plugins": {
    "claude-code-plugin-eslint": {
      "enabled": true,
      "config": {
        "rules": ["react", "typescript"],
        "autoFix": true
      }
    }
  }
}
```

`plugins` completa el ecosistema de extensiones registrando modulos que amplian las capacidades del agente con funcionalidades especializadas. A diferencia de los hooks, que responden a eventos del ciclo de vida del agente, los plugins pueden introducir comandos nuevos, modificar el comportamiento de herramientas existentes o anadir fuentes de contexto adicional. La configuracion de cada plugin es especifica del modulo y se declara dentro de su bloque, permitiendo activar o desactivar componentes individualmente sin alterar el resto del sistema. Entre los plugins mas comunes se encuentran integraciones con linters (ESLint, Ruff), generadores de documentacion, analizadores de dependencias y conectores con sistemas de incidencias como Jira o Linear.

```json
{
  "model": "claude-sonnet-4-20250624",
  "agent": "mi-agente-personalizado",
  "allowedTools": ["Read", "Edit", "Bash", "Glob", "Grep"],
  "disallowedTools": [],
  "permissions": {
    "additionalDirectories": ["/opt/data"]
  },
  "advisorModel": "claude-opus-4-20260514",
  "autoMode": {
    "enabled": true,
    "classifierThreshold": 0.88
  },
  "theme": "dracula",
  "outputStyle": "markdown",
  "keybindings": {
    "ctrl+p": "comando.plan",
    "ctrl+r": "comando.review"
  },
  "hooks": {
    "PreToolUse": {
      "command": "scripts/hook-pre.js",
      "timeout": 3000
    },
    "PostToolUse": {
      "command": "scripts/hook-post.js"
    }
  },
  "mcpServers": {
    "mi-servidor": {
      "command": "node",
      "args": ["server.js"],
      "env": {
        "API_KEY": "${API_KEY}"
      }
    }
  },
  "plugins": {
    "mi-plugin": {
      "enabled": true
    }
  }
}
```

> La clave `agent` apunta a un agente definido en el archivo `CLAUDE.md` del proyecto o en la configuracion global. Si no se especifica, Claude Code utiliza el agente por defecto del modelo activo.

La comprension de `settings.json` como sistema de capas -- proyecto, usuario, managed -- permite disenar entornos de desarrollo donde la flexibilidad individual no compromete la gobernanza corporativa. Un desarrollador puede personalizar su tema y sus atajos de teclado sin poder eludir las restricciones de herramientas peligrosas que la organizacion ha bloqueado via managed settings, y un equipo puede estandarizar el modelo y los hooks de pre-commit sin impedir que cada miembro ajuste su experiencia visual. Esta arquitectura refleja el mismo principio que los sistemas de configuracion de editores como VS Code o Neovim, pero trasladado al plano operativo de un agente autonomo con capacidad de modificar el sistema de ficheros.

