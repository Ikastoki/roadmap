# OpenCode CLI

La arquitectura de agentes en OpenCode permite desacoplar roles, Modelos de Lenguaje de Gran Tamano (LLM) y permisos en identidades independientes que colaboran dentro de una misma sesion. Cada agente encapsula un sistema de prompt, un modelo, una temperatura y un conjunto de herramientas habilitadas, lo que posibilita alternar entre modos de trabajo -- desde la exploracion exploratoria de un codigo base hasta la escritura directa de archivos -- sin necesidad de reconfigurar manualmente el entorno. Este diseno resulta especialmente valioso en flujos donde distintos tipos de intervencion (analisis, implementacion, revision, documentacion) requieren comportamientos y restricciones distintos.

## Agentes integrados

OpenCode incluye cinco agentes de uso general y tres agentes internos de sistema. Los dos agentes primarios son `build` y `plan`. `build` es el agente por defecto: dispone de todas las herramientas habilitadas (lectura, escritura, edicion, ejecucion de comandos, busqueda, navegacion web) y representa el perfil adecuado para tareas de desarrollo que requieren intervencion completa sobre el codigo base y el sistema de archivos. `plan`, en cambio, opera en modo exclusivamente analitico: sus herramientas de escritura y ejecucion de comandos estan configuradas con el valor `ask`, de modo que cualquier modificacion potencial requiere confirmacion explicita del usuario. Esto lo convierte en la opcion idonea para revisiones de codigo, diseno de arquitectura o planeacion previa a la implementacion.

Junto a los primarios, OpenCode incorpora tres subagentes. `general` es un subagente de proposito general con acceso completo a herramientas (excepto `todowrite`), disenado para ejecutar subtareas complejas de forma autonoma y en paralelo. `explore` es un agente de solo lectura especializado en la exploracion del codigo base: puede leer archivos, ejecutar busquedas con glob y grep, y listar directorios, pero nunca modifica ficheros ni ejecuta comandos arbitrarios. `scout` es igualmente de solo lectura pero esta orientado a la documentacion externa y dependencias: puede clonar repositorios en la cache gestionada por OpenCode, inspeccionar librerias de terceros y cruzar referencias con el codigo local.

Los agentes internos -- `compaction`, `title` y `summary` -- son agentes primarios ocultos que el sistema invoca automaticamente. `compaction` se encarga de comprimir el contexto de la conversacion cuando este supera el limite del modelo, generando un resumen compacto que preserva la informacion esencial. `title` genera titulos descriptivos para cada sesion. `summary` produce resumenes de sesion al finalizar. Estos agentes no son seleccionables en la interfaz de usuario y su ejecucion es completamente transparente para el operador.

## Tipos de agentes

OpenCode distingue dos categorias fundamentales: agentes primarios y subagentes. Los agentes primarios son aquellos con los que el usuario interactua directamente; se pueden ciclar durante una sesion mediante la tecla Tab o el atajo configurado `switch_agent`. Cada sesion tiene un agente primario activo que es el destinatario de los mensajes del usuario. Los subagentes, por el contrario, son invocados automaticamente por el agente primario cuando este detecta que una tarea se ajusta a la descripcion de un subagente disponible, o manualmente mediante la mencion `@` en el mensaje. Por ejemplo, escribir `@explore busca todas las funciones que usan async` invoca al subagente `explore` directamente.

La propiedad `mode` determina como puede utilizarse un agente. Los valores posibles son `primary`, `subagent` y `all`. El valor `primary` restringe el agente al ciclo de Tab; `subagent` lo limita a invocacion mediante `@` o a traves de la herramienta `task`; `all` permite ambos usos. Si no se especifica `mode`, el valor por defecto es `all`. El enrutamiento de un mensaje hacia un subagente ocurre cuando el modelo del agente primario decide, durante su ciclo de razonamiento, que una subtarea se beneficiaria de un perfil diferente: ya sea por restricciones de permisos, por especializacion del prompt o por conveniencia de aislamiento de contexto. Los subagentes ejecutan su tarea en una sesion hija y devuelven el resultado al agente primario, manteniendo la sesion principal limpia de detalle operativo.

## Configuracion de agentes

La configuracion de agentes puede realizarse de dos formas: mediante el archivo `opencode.json` (formato JSON) o mediante archivos Markdown con frontmatter YAML ubicados en los directorios `~/.config/opencode/agents/` (global) o `.opencode/agents/` (por proyecto). El nombre del archivo Markdown (sin la extension `.md`) se convierte en el identificador del agente. Por ejemplo, `revisor.md` crea un agente invocable como `@revisor`.

Las opciones de configuracion incluyen:

- `description` (obligatoria): texto breve que describe el proposito del agente. Se muestra en la interfaz y se expone al LLM para que este pueda decidir cuando delegar una tarea.
- `prompt`: archivo de sistema de prompt personalizado, referenciado con la sintaxis `{file:./ruta/al/prompt.txt}`. La ruta es relativa a la ubicacion del archivo de configuracion.
- `model`: identificador del modelo en formato `proveedor/modelo`. Si no se especifica, los agentes primarios heredan el modelo global configurado y los subagentes heredan el modelo del agente que los invoca.
- `temperature`: valor entre 0.0 y 1.0 que controla la aleatoriedad de las respuestas. Valores bajos (0.0-0.2) producen respuestas deterministicas; valores altos (0.6-1.0) favorecen la creatividad.
- `top_p`: alternativa a temperature para controlar la diversidad, tambien en rango 0.0 a 1.0.
- `permission`: objeto que asigna a cada herramienta (o grupo de herramientas) un valor `allow`, `ask` o `deny`. Las claves disponibles son `read`, `edit` (cubre `write`, `edit`, `apply_patch`), `glob`, `grep`, `list`, `bash`, `task`, `external_directory`, `todowrite`, `webfetch`, `websearch`, `lsp`, `skill`, `question` y `doom_loop`. Las claves `bash`, `edit`, `read`, `glob`, `grep`, `list`, `task`, `external_directory`, `lsp` y `skill` aceptan tanto el valor directo como un objeto de mapeo glob -> accion, permitiendo un control granular. Por ejemplo, `bash` puede configurarse para permitir `git status` sin confirmacion pero requerir confirmacion para cualquier otro comando.
- `tools` (deprecado): alternativa booleana anterior al sistema de permisos. `true` equivale a `{"*": "allow"}` y `false` a `{"*": "deny"}`.
- `mode`: `primary`, `subagent` o `all`.
- `steps`: numero maximo de iteraciones agente que el agente puede realizar antes de verse forzado a responder solo con texto.
- `color`: color hexadecimal o nombre de color del tema para personalizar la apariencia en la interfaz.
- `hidden`: cuando es `true`, oculta el subagente del menu de autocompletado `@`. Solo aplica a agentes con `mode: subagent`.
- `disable`: cuando es `true`, deshabilita el agente por completo.
- `task`: controla que subagentes puede invocar un agente mediante la herramienta `task`. Usa patrones glob. El orden de evaluacion importa: la ultima regla que coincide es la que prevalece.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "build": {
      "mode": "primary",
      "permission": {
        "edit": "allow",
        "bash": {
          "*": "ask",
          "git status*": "allow",
          "npm test*": "allow"
        }
      }
    },
    "plan": {
      "mode": "primary",
      "permission": {
        "edit": "deny",
        "bash": "deny"
      }
    },
    "code-reviewer": {
      "description": "Revisa codigo en busca de problemas de seguridad y calidad",
      "mode": "subagent",
      "temperature": 0.1,
      "permission": {
        "edit": "deny",
        "bash": {
          "*": "deny",
          "git diff*": "allow",
          "git log*": "allow"
        }
      }
    }
  }
}
```

## Agentes personalizados

OpenCode permite crear agentes personalizados mediante el comando interactivo `opencode agent create`. Este comando guia al usuario a traves de cinco pasos: seleccion de ubicacion (global o proyecto), descripcion del agente, generacion automatica de un identificador y un prompt de sistema adecuados, seleccion de los permisos que se concederan (todo lo no seleccionado se deniega implicitamente) y creacion del archivo Markdown resultante. El archivo generado sigue el formato de frontmatter YAML, donde las propiedades de configuracion se declaran entre bloques `---` y el cuerpo Markdown constituye el sistema de prompt.

Los agentes personalizados se almacenan como archivos Markdown en `~/.config/opencode/agents/` (disponibles globalmente) o `.opencode/agents/` (disponibles solo en el proyecto). Cuando existe un agente con el mismo nombre en ambos niveles, la configuracion del proyecto prevalece sobre la global. El nombre del archivo determina el identificador del agente y debe usar minusculas, guiones y caracteres alfanumericos.

A continuacion se muestra un ejemplo completo de un agente personalizado para revision de codigo:

```yaml
---
description: Revisor de codigo especializado en seguridad y rendimiento
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.1
steps: 30
color: "#e74c3c"
permission:
  edit: deny
  bash:
    "*": deny
    "git diff*": allow
    "git log*": allow
    "grep *": allow
  webfetch: deny
  websearch: deny
---

Eres un revisor de codigo experto en seguridad, rendimiento y mantenibilidad.

Directrices:
- Identifica vulnerabilidades de seguridad: inyeccion SQL, XSS, desbordamiento de buffer, exponer secretos.
- Seniala problemas de rendimiento: bucles innecesarios, consultas N+1, falta de indices, fugas de memoria.
- Evalua la mantenibilidad: complejidad ciclomatica, acoplamiento excesivo, falta de cohesion.
- Revisa el cumplimiento de las convenciones del proyecto y las buenas practicas del lenguaje.
- Proporciona sugerencias concretas con fragmentos de codigo corrigiendo los problemas detectados.
- No modifiques ningun archivo directamente. Limitate a analizar y reportar hallazgos.

Formato de respuesta:
1. Resumen ejecutivo de los hallazgos.
2. Lista detallada con gravedad (critica, alta, media, baja), ubicacion y recomendacion.
3. Conclusion con las acciones prioritarias.
```

El sistema de frontmatter YAML acepta cualquier clave adicional que no este definida en el esquema, la cual se pasa directamente al proveedor como opciones del modelo. Esto permite utilizar caracteristicas especificas de cada proveedor, como `reasoningEffort` en modelos de OpenAI o `textVerbosity` en otros proveedores. Ademas, el campo `prompt` puede omitirse en el frontmatter cuando el cuerpo Markdown del archivo contiene el sistema de prompt, que es precisamente el caso de los agentes definidos como archivos `.md`. En configuraciones JSON, en cambio, el prompt se especifica mediante la clave `prompt` y puede referenciar un archivo externo con la sintaxis `{file:./ruta/al/prompt.txt}`.

Los agentes personalizados heredan por defecto los permisos globales del archivo `opencode.json`, pero pueden sobrescribirlos parcial o totalmente. La combinacion de `description`, `permission` y `prompt` especifica define perfiles altamente especializados: un auditor de seguridad con acceso solo de lectura, un generador de documentacion con capacidad de escritura pero sin ejecucion de comandos, o un arquitecto de sistemas con permisos para ejecutar comandos de diagnostico pero no para modificar archivos. Esta granularidad convierte al sistema de agentes en una herramienta de orquestacion de flujos de trabajo donde cada identidad opera dentro de los limites que su funcion requiere.

