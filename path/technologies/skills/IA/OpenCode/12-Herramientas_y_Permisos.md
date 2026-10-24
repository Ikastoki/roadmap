# OpenCode CLI

## Herramientas integradas

OpenCode expone a los agentes un conjunto de herramientas nativas que cubren las operaciones fundamentales necesarias para interactuar con el sistema de archivos, ejecutar comandos, buscar informacion y coordinar trabajo asincrono. Cada herramienta esta disenada para un proposito especifico y se invoca automaticamente cuando el modelo determina que es la accion adecuada para el contexto actual. Comprender el funcionamiento y las limitaciones de cada una es esencial tanto para escribir configuraciones de permisos precisas como para disenar agentes que aprovechen las capacidades del sistema de forma eficiente.

La herramienta `read` permite leer archivos del sistema de forma segmentada: admite parametros de inicio (`offset`) y limite (`limit`) de lineas, lo que evita consumir contexto innecesariamente en ficheros extensos. Es la herramienta de lectura preferida frente a `bash cat` porque mantiene el control dentro del gestor de herramientas y permite un seguimiento fino del contenido consumido por el modelo. Su contrapartida, `write`, escribe ficheros completos. Siempre sobrescribe el destino y, en el caso de archivos existentes, exige que el modelo haya leido previamente el contenido para evitar perdidas accidentales de informacion. La herramienta `edit` realiza sustituciones de cadenas exactas dentro de un archivo: recibe un `oldString` y un `newString`, localiza la primera coincidencia unica del texto original y la reemplaza. Si el patron aparece multiples veces sin contexto suficiente, la operacion falla, lo que fuerza al modelo a proporcionar un fragmento de contexto mayor que desambigue la ubicacion. Adicionalmente, `patch` permite aplicar diffs unificados, util para cambios estructurales que afectan multiples lineas sin requerir multiples invocaciones de `edit`.

La busqueda de archivos y contenido se cubre con `glob` y `grep`. `glob` resuelve patrones de busqueda de ficheros (`**/*.ts`, `src/**/*.py`) y devuelve las rutas coincidentes. Esta herramienta es la alternativa correcta al comando `find` del sistema, ya que opera sobre el mismo indice que el resto de herramientas y no depende del shell subyacente. `grep` realiza busquedas de contenido con expresiones regulares completas en el interior de los ficheros, aceptando filtros por tipo de archivo y directorio. Ambas herramientas pueden encadenarse: una busqueda `glob` seguida de multiples `grep` constituye el patron tipico para localizar puntos de codigo antes de una edicion.

La ejecucion de comandos del sistema se realiza mediante la herramienta `bash`, que ejecuta comandos en un shell persistente. Acepta un parametro `command` con la cadena a ejecutar y un `workdir` opcional que define el directorio de trabajo, eliminando la necesidad de anteponer `cd` a cada instruccion. La sesion de shell se mantiene activa entre invocaciones, lo que permite conservar variables de entorno, cambios de directorio y estados. Es la herramienta con mayor superficie de riesgo: cualquier comando del sistema puede ejecutarse, desde lecturas de ficheros hasta modificaciones del sistema. Por esta razon, el sistema de permisos trata `bash` con un nivel de granularidad superior al del resto de herramientas, permitiendo patrones glob en los comandos.

Para acceso a informacion externa, OpenCode proporciona `fetch` y `websearch`. `fetch` realiza peticiones HTTP a URLs concretas y devuelve el contenido en formato texto, markdown o HTML segun se especifique. Es util para consultar APIs REST, leer documentacion en linea o verificar el estado de servicios remotos. `websearch` ejecuta busquedas en el proveedor configurado (como Brave o Google) y devuelve resultados web procesados. El numero de resultados y la profundidad del rastreo se configuran desde `opencode.json`, al igual que el modo de rastreo en vivo (`livecrawl`), que prioriza la obtencion de contenido fresco frente a resultados en cache.

La orquestacion de trabajo complejo se apoya en `task` y `todowrite`. `task` lanza subagentes independientes que ejecutan un objetivo con un conjunto de herramientas propio, permitiendo paralelizar busquedas o ediciones en multiples ficheros sin bloquear el flujo principal. Cada subagente opera en su propio contexto y devuelve un resultado estructurado al agente padre. `todowrite` gestiona listas de tareas estructuradas: permite definir objetivos, marcar tareas como completadas y mantener un estado visible que persiste a lo largo de la sesion. Esta herramienta es especialmente util en sesiones largas donde el modelo necesita recordar que pasos ha completado y cuales estan pendientes.

Dos herramientas adicionales cierran el ecosistema: `lsp` y `skill`. `lsp` se conecta a servidores de lenguaje (Language Server Protocol) para obtener diagnosticos, completado de codigo, definiciones y referencias. Cuando OpenCode detecta un proyecto con un LSP configurado (por ejemplo, `typescript-language-server` o `pyright`), la herramienta puede consultar errores de compilacion, saltar a la definicion de un simbolo o solicitar sugerencias contextuales sin ejecutar comandos externos. `skill` carga instrucciones reutilizables empaquetadas como skills: conjuntos de reglas, ejemplos y flujos de trabajo que especializan al agente para tareas repetitivas. A diferencia de los agentes, que son configuraciones completas, una skill es un fragmento de instrucciones que se inyecta en la solicitud del sistema cuando se activa.

## Sistema de permisos

El sistema de permisos de OpenCode constituye la capa de seguridad que media entre las intenciones del modelo y las acciones reales sobre el sistema. Sin permisos, un agente podria ejecutar cualquier comando, leer o escribir cualquier archivo y realizar peticiones de red sin restriccion. El sistema resuelve este problema definiendo tres niveles de autorizacion: `allow`, que ejecuta la accion sin intervencion del usuario; `ask`, que detiene la ejecucion y muestra la accion propuesta en la interfaz TUI para que el usuario la apruebe o rechace; y `deny`, que bloquea la accion directamente sin notificacion al usuario. El nivel por omision para herramientas no configuradas es `ask`, lo que garantiza que ninguna operacion nueva pase desapercibida.

La configuracion de permisos se expresa en `opencode.json` con una estructura jerarquica que permite distintos niveles de granularidad. Para `bash`, los permisos pueden definirse por patron glob sobre el comando: `"bash: \"npm *\"": "allow"` permite cualquier comando que empiece por `npm ` sin preguntar, mientras que `"bash: \"rm -rf /*\"": "deny"` bloquea una operacion destructiva. Esta granularidad por patron es exclusiva de `bash` debido a su naturaleza potencialmente peligrosa; para el resto de herramientas, los permisos se aplican globalmente al nombre de la herramienta (por ejemplo, `"read": "allow"`, `"write": "ask"`).

El ambito de los permisos tambien puede segmentarse por agente y por herramienta MCP. En la seccion `agent` de la configuracion, cada agente puede definir sus propias reglas de permiso que sobrescriben las globales. Del mismo modo, las herramientas expuestas por servidores MCP heredan el nivel de permiso definido para el servidor o pueden configurarse individualmente. Esta separacion permite, por ejemplo, que un agente dedicado a tareas de mantenimiento tenga acceso total a `bash` mientras que un agente de consulta tenga todas las herramientas en `deny` excepto `read` y `grep`.

La variable de entorno `OPENCODE_PERMISSION` permite configurar el nivel de permiso global de forma programatica. Su valor debe ser `allow`, `ask` o `deny` y sobrescribe cualquier configuracion en `opencode.json`. Esta variable esta disenada para entornos CI/CD donde no existe un usuario que pueda responder a las solicitudes de aprobacion: estableciendo `OPENCODE_PERMISSION=deny` se garantiza que el agente no ejecute ninguna accion potencialmente insegura, mientras que `OPENCODE_PERMISSION=allow` se utiliza en entornos controlados donde se confia plenamente en las instrucciones proporcionadas.

Cuando una herramienta configurada como `ask` esta a punto de ejecutarse, OpenCode pausa el flujo y presenta la accion propuesta en la interfaz TUI. El panel muestra el nombre de la herramienta, los parametros completos que recibira y el contexto de la conversacion que motivo la invocacion. El usuario dispone de las opciones para permitir una sola vez, permitir siempre, denegar una vez, denegar siempre o editar los parametros antes de la ejecucion. Esta ultima opcion permite, por ejemplo, modificar un comando `bash` peligroso antes de autorizarlo, transformando `rm -rf /tmp/project` en `rm -rf /tmp/project/build` sin necesidad de rechazar y reorientar al modelo.

```json
{
  "permissions": {
    "read": "allow",
    "write": "ask",
    "edit": "ask",
    "glob": "allow",
    "grep": "allow",
    "bash": {
      "npm *": "allow",
      "git *": "allow",
      "docker *": "ask",
      "rm -rf *": "deny",
      "sudo *": "deny",
      "*": "ask"
    },
    "fetch": "allow",
    "websearch": "allow"
  },
  "agents": {
    "mantenimiento": {
      "permissions": {
        "bash": "allow"
      }
    },
    "consulta": {
      "permissions": {
        "bash": "deny",
        "write": "deny",
        "edit": "deny"
      }
    }
  },
  "mcpServers": {
    "mi-servidor": {
      "command": "node",
      "args": ["server.js"],
      "permissions": "ask"
    }
  }
}
```

En el ejemplo anterior, las herramientas de lectura y busqueda operan sin restricciones, mientras que las de escritura requieren confirmacion. Los comandos `npm` y `git` se ejecutan sin preguntar, `docker` pide confirmacion, y cualquier comando con `rm -rf` o `sudo` queda bloqueado. El agente de mantenimiento tiene permiso total sobre `bash`, lo que permite automatizaciones de limpieza y despliegue sin intervencion. El agente de consulta solo puede leer y buscar, no modificar nada. Los servidores MCP, por su parte, heredan el nivel `ask` del bloque `mcpServers`, aunque podrian sobrescribirse por herramienta individual.

> La combinacion de permisos por patron en `bash` y por agente permite disenar flujos de trabajo donde un agente principal orquesta la sesion con permisos restrictivos mientras delega tareas concretas a subagentes con permisos mas amplios. Esta estrategia mantiene el principio de minimo privilegio sin sacrificar la automatizacion.

