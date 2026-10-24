# OpenCode CLI

## LSP

El Language Server Protocol (LSP) es un estandar abierto que define la comunicacion entre un editor o herramienta cliente y un servidor de lenguaje que proporciona funcionalidades de inteligencia de codigo: diagnostico de errores, navegacion a definiciones, busqueda de referencias, informacion contextual al situar el cursor y completado inteligente. OpenCode integra este protocolo para dotar al agente de comprension semantica del codigo que edita. Cuando la opcion `lsp: true` esta habilitada en la configuracion del proyecto, el agente activa servidores LSP para los lenguajes que reconoce y los utiliza como fuente de informacion en tiempo real durante las tareas de edicion y refactorizacion.

```json
{
  "lsp": true,
  "languages": {
    "typescript": {
      "server": "typescript-language-server",
      "args": ["--stdio"],
      "settings": {
        "typescript": {
          "suggest": {
            "autoImports": true
          }
        }
      }
    },
    "python": {
      "server": "pyright-langserver",
      "args": ["--stdio"],
      "settings": {
        "python": {
          "analysis": {
            "typeCheckingMode": "strict"
          }
        }
      }
    },
    "java": {
      "server": "eclipse-jdtls",
      "args": [
        "-configuration",
        "${workspace}/.jdtls/config",
        "-data",
        "${workspace}/.jdtls/data"
      ]
    }
  }
}
```

La arquitectura interna de la integracion LSP en OpenCode sigue el modelo clasico cliente-servidor JSON-RPC. Cuando el agente inicia una sesion de trabajo, OpenCode lanza el servidor LSP configurado como un subproceso independiente, establece los canales de comunicacion stdin/stdout y envia un mensaje `initialize` con las capacidades del cliente y la raiz del workspace. A partir de ese momento, el agente mantiene una sincronizacion continua del estado de los documentos abiertos: cada vez que el agente modifica un archivo, OpenCode notifica al servidor LSP mediante notificaciones `textDocument/didChange`, y el servidor responde con los diagnosticos actualizados a traves de notificaciones `textDocument/publishDiagnostics`. Estos diagnosticos incluyen la gravedad (error, warning, informacion), el rango afectado y un mensaje descriptivo que el modelo puede interpretar para corregir el problema automaticamente.

El agente consulta el servidor LSP antes de realizar modificaciones significativas. Por ejemplo, cuando necesita refactorizar una funcion, primero solicita `textDocument/references` para localizar todos los puntos de uso y evitar romper llamadas existentes. Del mismo modo, antes de anadir una nueva dependencia, utiliza `textDocument/completion` para verificar que el modulo esta disponible y que los tipos coinciden con lo esperado. Esta interaccion convierte al servidor LSP en una fuente de verdad constante que el modelo consulta de forma transparente, sin que el usuario perciba la complejidad subyacente del protocolo.

> Los servidores LSP pueden consumir recursos significativos, especialmente en proyectos grandes. OpenCode los gestiona como procesos longevos que permanecen activos durante toda la sesion, pero los termina automaticamente al cerrarse. Es posible deshabilitar LSP por completo con `lsp: false` si el proyecto no requiere analisis semantico o si los recursos del sistema son limitados.

## Plugins

El sistema de plugins de OpenCode permite extender las capacidades del agente mas alla de las herramientas integradas y los servidores MCP. Un plugin es un modulo npm que, una vez instalado, puede registrar herramientas personalizadas que el agente descubre e invoca durante la sesion de trabajo. El comando `opencode plugin <modulo>` instala el paquete desde el registro npm y lo vincula al proyecto, almacenando los archivos necesarios en el directorio `.opencode/plugins/`. Este directorio funciona como punto de carga: al iniciar una sesion, OpenCode escanea su contenido, importa cada modulo y ejecuta su rutina de inicializacion para registrar las herramientas que expone.

```bash
# Instalar un plugin desde npm
opencode plugin @opencode/plugin-docker
opencode plugin @opencode/plugin-database-explorer

# Listar plugins instalados
ls .opencode/plugins/
```

El ciclo de vida de un plugin comienza con la instalacion, durante la cual OpenCode descarga el paquete y lo coloca en `.opencode/plugins/`. En el arranque de la sesion, cada plugin se carga en un contexto aislado y se invoca su funcion de registro, que declara los nombres, esquemas y descripciones de las herramientas que proporciona. Cuando el agente decide utilizar una de esas herramientas, OpenCode ejecuta la funcion asociada en el contexto del plugin y devuelve el resultado al flujo de la conversacion. Si el plugin falla durante la carga, OpenCode lo omite y continua con el resto, registrando el error en los logs para facilitar la depuracion.

Los plugins pueden implementar practicamente cualquier funcionalidad que sea expresable como una operacion de entrada-salida. Algunos ejemplos representativos incluyen plugins para linters personalizados que envuelven herramientas como ESLint o Ruff y exponen sus reglas como herramientas invocables; plugins de despliegue que interactuan con plataformas como Vercel, Netlify o AWS para orquestar releases desde la conversacion; y plugins de exploracion de bases de datos que permiten al agente consultar esquemas, ejecutar queries supervisadas y visualizar resultados sin salir del terminal.

```json
{
  "plugins": {
    "opencode-docker": {
      "enabled": true
    },
    "opencode-linter": {
      "enabled": true,
      "config": {
        "rules": ["no-unused-vars", "no-console"]
      }
    }
  }
}
```

El ecosistema de plugins de OpenCode se encuentra en expansion continua. La comunidad publica plugins en el registro npm bajo el ambito `@opencode/plugin-*`, y es posible descubrirlos mediante `opencode plugin search <termino>`. La combinacion de LSP para inteligencia de codigo y plugins para funcionalidad extensible convierte a OpenCode en una plataforma de asistencia al desarrollo cuya superficie de capacidades crece con cada nuevo modulo que se integra.

