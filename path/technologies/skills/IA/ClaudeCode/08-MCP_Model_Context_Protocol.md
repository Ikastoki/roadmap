# Claude Code CLI

El Model Context Protocol (MCP) es un estandar abierto que define como los asistentes de IA pueden interactuar con herramientas y recursos externos de forma estructurada. Claude Code lo adopta como mecanismo de extension de primera clase: cualquier servidor compatible con MCP se convierte en una fuente de herramientas que el agente puede invocar durante la sesion de trabajo, exactamente igual que las herramientas integradas como `read`, `write` o `bash`. Esto convierte a MCP en el sistema de plugins oficial de Claude Code, permitiendo extender sus capacidades sin modificar el nucleo del producto.

## Configuracion MCP

La gestion de servidores MCP se realiza a traves del subcomando `claude mcp`. `claude mcp add` inicia un asistente interactivo que guia en la configuracion de un nuevo servidor, solicitando el tipo (local o remoto), el comando de ejecucion o la URL, y las variables de entorno necesarias. `claude mcp list` muestra todos los servidores configurados con su estado actual, indicando si estan conectados, si tienen errores de autenticacion o si estan desconectados. `claude mcp remove` elimina un servidor de la configuracion.

```bash
claude mcp add
claude mcp list
claude mcp remove my-server
```

Para servidores que requieren autenticacion OAuth, `claude mcp login` inicia el flujo de autorizacion abriendo el navegador para completar el proceso, mientras que `claude mcp logout` elimina las credenciales almacenadas. El comando slash `/mcp` abre un panel interactivo dentro de la sesion de Claude Code donde se pueden gestionar servidores, ver herramientas disponibles y depurar problemas de conexion sin salir del terminal.

```bash
claude mcp login my-server
claude mcp logout my-server
```

> El comando `claude mcp login` solo es necesario para servidores remotos que implementan el flujo de autorizacion OAuth del protocolo MCP. Los servidores locales con autenticacion por API key gestionan sus credenciales mediante variables de entorno.

## Servidores locales y remotos

Los servidores MCP se dividen en dos categorias segun su modelo de ejecucion. Los servidores locales utilizan el transporte `stdio`: Claude Code los ejecuta como subprocesos del sistema, comunicandose con ellos a traves de la entrada y salida estandar mediante mensajes JSON-RPC. Esto los hace ideales para herramientas que necesitan acceso directo al sistema de ficheros, ejecutar binarios locales o interactuar con servicios que se ejecutan en la maquina del desarrollador.

Los servidores remotos utilizan transporte HTTP con Server-Sent Events (SSE) para la comunicacion bidireccional. Claude Code se conecta a una URL, recibe eventos del servidor y envia solicitudes HTTP. Estos servidores pueden implementar autenticacion OAuth, permitiendo que multiples desarrolladores compartan el mismo servidor sin exponer credenciales individuales.

La configuracion MCP se almacena en el archivo `.mcp.json` en la raiz del proyecto, lo que permite compartir la configuracion con todo el equipo mediante control de versiones. Este archivo es versionable y sigue la misma logica de `CLAUDE.md`: cada miembro del equipo obtiene las mismas herramientas MCP al clonar el repositorio.

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem"],
      "env": {
        "WORKSPACE_ROOT": "/home/usuario/proyecto"
      }
    },
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "{env:GITHUB_TOKEN}"
      }
    },
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/sse",
      "headers": {
        "Authorization": "Bearer {env:SENTRY_TOKEN}"
      }
    },
    "monitoring": {
      "type": "remote",
      "url": "https://mcp.internal.company.com/sse",
      "headers": {
        "X-API-Key": "{env:INTERNAL_API_KEY}"
      }
    }
  }
}
```

> Las variables de entorno se referencian con la sintaxis `{env:VARIABLE}`. Claude Code resuelve estas referencias en tiempo de ejecucion, manteniendo las credenciales fuera del archivo de configuracion y permitiendo que cada desarrollador use sus propias claves sin modificar `.mcp.json`.

## Gestion y seguridad

Cada herramienta que expone un servidor MCP se integra en el sistema de permisos de Claude Code mediante un patron de nomenclatura especifico: `mcp__<nombre-del-servidor>__<nombre-de-la-herramienta>`. Esto permite un control granular sobre que herramientas puede utilizar el agente y en que contextos, definiendo permisos en `CLAUDE.md` o mediante el comando `/permissions`.

```bash
mcp__filesystem__read_allowed: true
mcp__github__create_issue: ask
mcp__sentry__*: deny
```

El patron `mcp__sentry__*` deniega todas las herramientas del servidor Sentry, mientras que `mcp__filesystem__read` permite el acceso directo al sistema de ficheros del proyecto. La combinacion de servidores locales y remotos con permisos granulares convierte a MCP en un sistema de extension seguro, donde cada herramienta tiene un alcance definido y el desarrollador mantiene control explicito sobre que capacidades se delegan al agente.

La integracion con herramientas externas abarca desde servicios de observabilidad como Sentry para consultar errores y trazas, hasta servidores de busqueda como `sequential-thinking` que implementan cadenas de razonamiento estructurado. Servidores como `filesystem` permiten a Claude Code operar sobre el sistema de archivos con operaciones que van mas alla de las herramientas integradas, mientras que servidores como `github` habilitan la gestion completa de issues, pull requests y workflows directamente desde la sesion conversacional.

El ecosistema MCP crece continuamente con servidores comunitarios y oficiales que cubren bases de datos, servicios cloud, APIs de producto y herramientas de infraestructura. La combinacion del transporte local para herramientas que requieren baja latencia y acceso al sistema, con servidores remotos para servicios centralizados, ofrece una arquitectura flexible que se adapta tanto a entornos de desarrollo individual como a equipos con infraestructura compartida.

