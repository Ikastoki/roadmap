# Claude Code CLI

La capacidad de delegar trabajo de forma automatica a agentes especializados es uno de los mecanismos internos mas relevantes de Claude Code. Cuando el agente principal identifica que una tarea puede fragmentarse en operaciones paralelas o requiere un ambito de ejecucion aislado, lanza subagentes que operan con su propio contexto, su propio sandbox de archivos y, opcionalmente, su propio modelo de lenguaje. Esto permite que mientras un subagente ejecuta una bateria de tests, otro pueda estar refactorizando un modulo y un tercero analizando dependencias, todo dentro de una misma sesion y sin que el usuario tenga que gestionar manualmente la concurrencia.

## Agentes integrados

El agente primario (primary agent) es el que recibe el prompt inicial del usuario y mantiene la conversacion principal. Cuando detecta que puede acelerar la resolucion de un problema delegando subtareas, crea subagentes. Estos subagentes no son procesos independientes, sino instancias hijas que heredan parte del contexto del padre pero operan con un alcance limitado a la tarea encomendada. Existen dos modalidades de ejecucion: los subagentes en primer plano (foreground) operan dentro de la misma conversacion y reportan sus resultados directamente en el hilo de dialogo, lo que resulta util para tareas cortas donde el usuario necesita ver el progreso; los subagentes en segundo plano (background) se ejecutan de forma autonomia y notifican al agente principal cuando finalizan, siendo adecuados para operaciones prolongadas que no requieren supervision inmediata.

> La diferencia fundamental entre foreground y background no es tecnica sino de visibilidad: ambos ejecutan codigo real y modifican archivos, pero los foreground exponen su output en la conversacion activa mientras que los background operan de forma silenciosa hasta completarse.

Desde la version 2.1.219, Claude Code introdujo el anidamiento de subagentes (nested subagents), permitiendo que un subagente pueda a su vez crear subagentes propios hasta una profundidad maxima de tres niveles. Esta capacidad resulta especialmente util en tareas de gran envergadura, como migraciones de codigo que requieren analisis, transformacion y verificacion en multiples capas. La profundidad maxima de anidamiento se controla mediante la variable de entorno `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`, que por defecto permite hasta tres niveles y puede reducirse para limitar el uso de recursos en entornos con restricciones de computo.

```bash
export CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=2
```

El uso de subagentes es transparente para el usuario final. Claude Code decide autonomamente cuando delegar trabajo a un subagente basandose en la estructura de la tarea, la complejidad estimada y la disponibilidad de recursos. En la interfaz TUI, la actividad de los subagentes se muestra en el panel lateral, permitiendo observar que tareas se estan ejecutando en paralelo y cual es su estado sin interferir con la conversacion principal.

## Creacion y configuracion

Aunque Claude Code utiliza subagentes internos de forma automatica, tambien permite definir subagentes personalizados que el usuario puede invocar explicitamente mediante el comando slash `/agent`. Estos subagentes se definen en archivos Markdown con frontmatter YAML ubicados en el directorio `.claude/agents/` del proyecto (alcance local) o en `~/.claude/agents/` (alcance global para todos los proyectos). Cada archivo representa un agente especializado con una personalidad, instrucciones y configuracion de modelo propias.

El frontmatter YAML admite multiples campos de configuracion. El campo `description` proporciona un resumen breve que aparece en la interfaz de seleccion; `prompt` contiene las instrucciones detalladas que definen el comportamiento del agente; `model` permite especificar un modelo distinto al del agente principal (por ejemplo, usar un modelo mas rapido y economico para tareas simples); y `temperature` controla la creatividad de las respuestas, con valores cercanos a 0 para tareas deterministicas y valores mas altos para generacion creativa. Ademas, pueden configurarse `max_tokens` para limitar la extension de las respuestas y `system` para instrucciones adicionales de sistema.

El comando `/agents` abre la interfaz grafica de gestion de subagentes dentro del TUI, donde es posible listar, activar, desactivar y eliminar agentes personalizados sin necesidad de editar archivos manualmente. Para invocar un subagente definido, se usa `/agent <nombre>` dentro de la sesion interactiva. A continuacion se muestra un ejemplo completo de un archivo de subagente para revision de codigo:

```yaml
---
description: >
  Revisor de codigo especializado en seguridad. Analiza el codigo fuente
  en busca de vulnerabilidades comunes, malas practicas de cifrado y
  fugas de informacion sensible.
prompt: >
  Eres un revisor de codigo experto en seguridad. Tu unica funcion es
  analizar el codigo que se te proporcione e identificar vulnerabilidades
  como inyeccion SQL, XSS, exposicion de secretos, uso inseguro de
  criptografia, desbordamiento de buffers y otros problemas de seguridad
  relevantes segun el lenguaje. Proporciona una lista priorizada de
  hallazgos con la linea exacta, una explicacion del riesgo y una
  sugerencia concreta de correccion. No modifiques ningun archivo ni
  ejecutes comandos fuera del analisis. Responde exclusivamente en
  espanol.
model: claude-sonnet-4-20250514
temperature: 0.1
max_tokens: 4096
system: >
  Eres una herramienta de revision de seguridad integrada en Claude Code.
  Manten un tono tecnico y directo. Si no encuentras vulnerabilidades,
  indica explicitamente que el codigo parece seguro. No inventes
  hallazgos.
---
```

Una vez guardado este archivo como `seguridad.md` en `.claude/agents/`, el subagente queda disponible para ser invocado con `/agent seguridad` dentro de cualquier sesion del proyecto. La combinacion de subagentes predefinidos por el sistema y agentes personalizados definidos por el usuario convierte a Claude Code en un sistema extensible donde cada equipo puede moldear el comportamiento de la herramienta para ajustarlo a sus flujos de trabajo, estandares de calidad y politicas de seguridad especificas.

