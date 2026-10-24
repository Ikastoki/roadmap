# OpenCode CLI

## Que es MCP

El Model Context Protocol (MCP) es un protocolo abierto y estandarizado que define como los agentes de inteligencia artificial pueden descubrir e invocar herramientas externas y fuentes de datos en tiempo de ejecucion. A diferencia de una integracion via API convencional, donde el desarrollador debe codificar explicitamente cada punto de conexion, MCP permite que el agente negocie capacidades con servidores externos de forma dinamica: el servidor publica un listado de herramientas disponibles con sus esquemas de entrada, y el agente decide que invocar segun la tarea que enfrenta.

El flujo comienza cuando OpenCode, durante la inicializacion de una sesion, carga la configuracion de servidores MCP definida en `opencode.json`. Para cada servidor configurado, el agente establece una conexion -- como subproceso local o mediante peticiones HTTP -- y solicita el listado de herramientas disponibles. Cada herramienta incluye su nombre, una descripcion legible para el modelo y un esquema JSON que define los parametros que acepta. Cuando el modelo determina que necesita una capacidad externa, genera una invocacion con los parametros adecuados, OpenCode la reenvia al servidor MCP correspondiente, y el resultado regresa al flujo de la conversacion como si hubiera sido generado por una herramienta nativa.

Esta arquitectura desacopla las capacidades del agente de su implementacion interna. Cualquier desarrollador puede escribir un servidor MCP que exponga herramientas especificas de su dominio -- una base de datos interna, un sistema de monitorizacion, un orquestador de contenedores -- y OpenCode las consumira sin modificaciones en su codigo base. El protocolo funciona sobre transporte JSON-RPC, lo que garantiza interoperabilidad entre lenguajes y plataformas.

## Servidores locales

Un servidor MCP local se ejecuta como un proceso hijo de OpenCode. Cuando el agente necesita una herramienta que expone un servidor local, OpenCode lanza el proceso especificado en la configuracion, se comunica con el mediante mensajes JSON-RPC sobre stdin/stdout, y lo mantiene vivo mientras sea necesario. Este modelo es ideal para herramientas que deben ejecutarse en el mismo entorno que el proyecto, como linters, generadores de documentacion o analizadores estaticos que necesitan acceso directo al sistema de archivos.

La configuracion de un servidor local se define bajo la clave `mcpServers` en `opencode.json`. El campo `type` debe ser `"local"`, y `command` es un array con el ejecutable y sus argumentos. Opcionalmente se pueden especificar variables de entorno con `environment`, el directorio de trabajo con `cwd` y un timeout en milisegundos con `timeout`.

```json
{
  "mcpServers": {
    "sentry": {
      "type": "local",
      "command": ["npx", "-y", "@sentry/mcp"],
      "environment": {
        "SENTRY_AUTH_TOKEN": "{env:SENTRY_AUTH_TOKEN}",
        "SENTRY_ORG": "mi-organizacion"
      },
      "cwd": "/ruta/del/proyecto",
      "timeout": 30000
    }
  }
}
```

El ciclo de vida de un servidor local esta ligado a la sesion de OpenCode. Se instancia bajo demanda cuando el agente decide invocar una de sus herramientas y se termina al cerrar la sesion o tras un periodo de inactividad superior al timeout configurado. La comunicacion via stdio elimina la necesidad de gestionar puertos de red, manteniendo la隔离 entre proyectos y evitando conflictos con otros servicios del sistema.

> Los servidores locales heredan las variables de entorno del proceso padre de OpenCode, pero las definidas en `environment` tienen prioridad. Es recomendable usar referencias a variables de entorno del sistema mediante `{env:VARIABLE}` en lugar de hardcodear credenciales en el archivo de configuracion.

## Servidores remotos

Un servidor MCP remoto se comunica con OpenCode mediante peticiones HTTP en lugar de un subproceso local. Esta modalidad es adecuada para servicios compartidos que deben estar accesibles desde multiples entornos o equipos: sistemas de monitorizacion centralizados, APIs de terceros, bases de datos corporativas o cualquier servicio que no tenga sentido ejecutar localmente.

La configuracion para un servidor remoto utiliza `type: "remote"`, la `url` del endpoint y opcionalmente un mapa de `headers` para cabeceras HTTP personalizadas. OpenCode soporta autenticacion OAuth automatica: cuando el servidor lo requiere, el agente inicia un flujo de autorizacion que el usuario completa en el navegador, y OpenCode almacena localmente el token resultante.

```json
{
  "mcpServers": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/sse",
      "headers": {
        "Authorization": "Bearer {env:CONTEXT7_API_KEY}"
      }
    },
    "grep-vercel": {
      "type": "remote",
      "url": "https://mcp.grep.vercel.ai/sse"
    }
  }
}
```

> Los servidores remotos deben considerarse entornos potencialmente no confiables. OpenCode ejecuta las herramientas que exponen, por lo que es fundamental verificar que el servidor pertenece a una fuente legitima y que las credenciales utilizadas tienen los permisos minimos necesarios.

La autenticacion OAuth se gestiona mediante el subcomando `opencode mcp auth`, que inicia el flujo de autorizacion en el navegador del sistema. Una vez completado, OpenCode persiste el token de refresco y lo renueva automaticamente cuando expira, sin intervencion del usuario. Para servidores que requieren una API key estatica, se configura mediante `headers` en combinacion con variables de entorno.

## Gestion de MCP

OpenCode proporciona varios subcomandos dentro de `opencode mcp` para administrar el ecosistema de servidores. `opencode mcp add` inicia un asistente interactivo que guia al usuario en la configuracion: solicita el tipo de servidor (local o remoto), el comando o URL segun corresponda, las variables de entorno necesarias y, para servidores locales, el timeout. Al finalizar, escribe o actualiza la seccion `mcpServers` en `opencode.json`.

`opencode mcp list` (o `mcp ls`) muestra el estado de todos los servidores configurados: nombre, tipo, si estan activos o detenidos, y la cantidad de herramientas que exponen. Es util para verificar rapidamente que la configuracion es correcta antes de iniciar una sesion de trabajo. Cuando un servidor falla al conectar, el listado muestra el error, facilitando la depuracion.

El control de acceso a las herramientas MCP se integra con el sistema de permisos de OpenCode. Es posible restringir que agentes pueden utilizar que herramientas MCP mediante patrones glob en la configuracion del agente. Por ejemplo, un agente "plan" puede tener acceso solo a herramientas de lectura como Sentry (consulta de errores), mientras que un agente "build" puede invocar todas las herramientas MCP disponibles.

```json
{
  "agent": {
    "plan": {
      "permission": {
        "mcp:read": "allow",
        "mcp:write": "deny"
      }
    }
  }
}
```

Entre los servidores MCP mas utilizados en el ecosistema OpenCode destacan Sentry MCP, que permite al agente consultar errores, crear issues y explorar el rendimiento de aplicaciones directamente desde la conversacion; Context7, que proporciona acceso a documentacion tecnica actualizada de librerias y frameworks; y Grep by Vercel, un motor de busqueda de codigo que opera sobre repositorios publicos de GitHub. Estos ejemplos ilustran como MCP transforma a OpenCode de un agente limitado al sistema de archivos local a un asistente capaz de interactuar con todo el ecosistema de desarrollo.

