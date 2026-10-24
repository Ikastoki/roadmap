# Claude Code CLI

## Modos de permiso

Claude Code expone tres modos de permiso que determinan el nivel de autonomia del agente para ejecutar herramientas y modificar el sistema de archivos. El modo `plan` es un modo exclusivamente de lectura: Claude puede leer archivos, navegar por el arbol de directorios y analizar el codigo, pero no puede escribir en disco ni ejecutar comandos. Este modo esta pensado para fases de analisis, revision de codigo o sesiones exploratorias donde se busca entender el estado del proyecto sin riesgo de introducir cambios. El modo `auto` utiliza un clasificador automatico basado en aprendizaje automatico que decide, accion por accion, si una operacion es lo suficientemente segura como para ejecutarse sin confirmacion o si requiere intervencion del usuario. El clasificador considera el tipo de herramienta, los argumentos involucrados y el contexto de la sesion para tomar la decision.

El modo `bypassPermissions` suprime por completo todas las confirmaciones: Claude ejecuta cualquier herramienta sin preguntar. Es el modo mas peligroso y debe reservarse exclusivamente para entornos controlados como pipelines de integracion continua, entornos de pruebas efimeros o escenarios donde se ha verificado manualmente que el agente no tiene acceso a recursos sensibles. El flag `--dangerously-skip-permissions` es un alias directo de este modo y su nombre explicito no es casual.

```bash
claude --permission-mode plan
claude --permission-mode auto
claude --permission-mode bypassPermissions
claude --dangerously-skip-permissions
```

El cambio entre modos durante una sesion interactiva se realiza pulsando Shift+Tab, que cicla secuencialmente entre `plan`, `auto` y `bypassPermissions`. El modo activo se muestra en la barra de estado del TUI, lo que permite al usuario verificar en todo momento el nivel de permisos vigente sin necesidad de interrumpir el flujo de trabajo.

## Sistema de permisos

Mas alla de los modos globales, Claude Code ofrece un sistema granular de permisos basado en listas de herramientas permitidas y denegadas. La configuracion `allowedTools` define el conjunto de herramientas que el agente puede ejecutar sin solicitar confirmacion, independientemente del modo de permiso activo. La sintaxis admite tres variantes: para comandos Bash especificos se usa el patron `"Bash(git log *)"`, que permite ejecutar cualquier comando que comience por `git log` sin preguntar; para habilitar una herramienta completa se usa el nombre simple como `"Read"` o `"Glob"`; y para herramientas MCP se emplea el patron `"mcp__github__*"`, donde el doble guion bajo separa el prefijo MCP del nombre del plugin y de la herramienta concreta.

```json
{
  "allowedTools": [
    "Bash(git log *)",
    "Bash(git diff *)",
    "Bash(git status)",
    "Read",
    "Glob",
    "Grep",
    "mcp__github__*"
  ],
  "disallowedTools": [
    "Bash(rm *)",
    "Bash(> *)",
    "Write(/etc/*)"
  ]
}
```

El campo `disallowedTools` sigue exactamente la misma sintaxis pero con semantica inversa: las herramientas o patrones listados se deniegan automaticamente, sin posibilidad de confirmacion por parte del usuario. Ambos campos admiten patrones glob, lo que permite coincidencias flexibles. Por ejemplo, `"Bash(sudo *)"` deniega cualquier comando que comience con `sudo`, mientras que `"Write"` denegaria toda escritura en el sistema de archivos. La combinacion de ambas listas permite construir perfiles de seguridad muy precisos: se puede permitir `"Bash(*)"` pero denegar `"Bash(sudo *)"`, o permitir `"Write"` pero denegar `"Write(/etc/*)"`.

El flag `--add-dir` extiende el alcance del agente a directorios fuera del proyecto. Por defecto, Claude Code solo puede leer y escribir dentro del directorio de trabajo. Con `--add-dir` se conceden accesos adicionales que se suman al directorio base, utiles cuando el proyecto depende de configuraciones o recursos ubicados en otras partes del sistema de archivos.

```bash
claude --add-dir /etc/nginx --add-dir /var/log
```

> Las rutas anadidas con `--add-dir` deben concederse explicitamente cada vez que se inicia la sesion; no persisten entre ejecuciones. Para accesos permanentes, es preferible configurarlos en el archivo `settings.json` del proyecto.

## Hooks

El sistema de hooks de Claude Code permite ejecutar scripts personalizados antes y despues de cada uso de herramienta, ofreciendo un mecanismo de extension para flujos de seguridad, auditoria y automatizacion. Los hooks `preToolUse` se ejecutan inmediatamente antes de que el agente invoque una herramienta, y pueden modificar los parametros de la llamada, bloquear la ejecucion o ejecutar acciones preparatorias. Los hooks `postToolUse` se ejecutan tras completarse la herramienta, con acceso al resultado de la misma.

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "Bash(*)",
        "script": "scripts/validate_command.sh",
        "block": true
      }
    ],
    "postToolUse": [
      {
        "matcher": "Write(*)",
        "script": "scripts/notify_changed_file.sh"
      }
    ]
  }
}
```

La configuracion de hooks reside en el archivo `settings.json` del proyecto y cada hook define un `matcher` (un patron glob sobre el nombre de la herramienta), un `script` (la ruta al script a ejecutar) y un flag `block` opcional que, cuando es `true`, impide la ejecucion de la herramienta si el script retorna un codigo distinto de cero. Esta capacidad de bloqueo convierte los hooks `preToolUse` en un mecanismo de seguridad detallado: se puede, por ejemplo, inspeccionar el comando bash que Claude intenta ejecutar, validar que no accede a recursos prohibidos y, si algo sospechoso se detecta, impedir la operacion.

Los hooks son especialmente utiles en entornos compartidos o en pipelines de CI donde se necesita garantizar que Claude opera dentro de unos limites definidos. Un caso de uso comun es ejecutar un linter de seguridad antes de cada comando bash, o enviar una notificacion a un canal de Slack cada vez que se modifica un archivo critico. La combinacion de hooks con las listas `allowedTools` y `disallowedTools` constituye un sistema de defensa en profundidad que permite desde entornos completamente restrictivos hasta flujos totalmente automatizados, adaptandose al nivel de confianza que se deposita en el agente.

```bash
# Ejemplo de script de validacion para preToolUse
#!/bin/bash
if echo "$1" | grep -q "sudo\|chmod 777\|/etc/shadow"; then
  echo "Comando bloqueado por seguridad"
  exit 1
fi
exit 0
```

