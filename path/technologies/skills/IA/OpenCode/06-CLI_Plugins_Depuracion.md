# OpenCode CLI

La arquitectura de OpenCode esta disenada para ser extensible desde el primer momento. No se trata de un binario monolitico con un conjunto fijo de capacidades, sino de un nucleo que puede ampliarse mediante un sistema de plugins, un protocolo de comunicacion entre agentes (ACP) que permite la interoperabilidad con herramientas externas, y un conjunto de comandos de depuracion que ofrecen visibilidad total sobre el estado interno del sistema. Estos tres mecanismos --extension, integracion y diagnostico-- conforman la capa mas avanzada del CLI, dirigida a usuarios que necesitan adaptar OpenCode a flujos de trabajo especificos, conectarlo con otros sistemas o resolver problemas de configuracion complejos.

## opencode plugin / plug

El sistema de plugins permite extender las capacidades de OpenCode mas alla de las herramientas integradas. Un plugin es un paquete npm que, una vez instalado, puede registrar nuevas herramientas, modificar el comportamiento del agente, agregar comandos personalizados al CLI o alterar el pipeline de procesamiento de mensajes. La instalacion se realiza mediante el subcomando `plugin` (o su alias `plug`) seguido del nombre del modulo npm:

```bash
# Instalar un plugin desde npm
opencode plugin opencode-plugin-mitool

# Instalar globalmente (disponible en todos los proyectos)
opencode plugin opencode-plugin-mitool --global

# Forzar la sustitucion de una version existente
opencode plugin opencode-plugin-mitool --force
```

El flag `--global` (o `-g`) instala el plugin en el directorio de datos global de OpenCode, haciendolo accesible desde cualquier proyecto. Sin esta bandera, el plugin se instala localmente dentro del directorio `.opencode/plugins/` del proyecto actual, lo que permite que cada proyecto tenga su propio conjunto de extensiones sin interferencias. El flag `--force` (o `-f`) sobrescribe una instalacion existente, util durante el desarrollo del plugin o cuando se necesita actualizar a una version especifica sin pasar por el gestor de paquetes.

Internamente, OpenCode descarga el paquete npm, lo instala en la ubicacion correspondiente y lo carga en el arranque. Los plugins pueden declarar en su `package.json` los ganchos (hooks) que implementan: por ejemplo, `onToolCall` para interceptar o extender llamadas a herramientas, `onMessage` para transformar mensajes antes de enviarlos al modelo, o `onCommand` para agregar subcomandos al CLI. La deteccion de plugins ocurre en la fase de inicializacion, antes de que el agente comience a procesar la sesion.

> El flag `--pure` del CLI desactiva la carga de todos los plugins, permitiendo ejecutar OpenCode sin extensiones de terceros. Es util para aislar problemas y verificar si un comportamiento inesperado es causado por un plugin.

## opencode acp

El subcomando `acp` convierte a OpenCode en un servidor del protocolo ACP (Agent Client Protocol). ACP es un protocolo de comunicacion entre agentes basado en JSON delimitado por nuevas lineas (nd-JSON) que utiliza stdin y stdout como canal de transporte. A diferencia de MCP (que conecta agentes con herramientas), ACP esta disenado para la comunicacion bidireccional entre agentes: un proceso puede enviar mensajes a OpenCode y recibir sus respuestas de forma estructurada, lo que permite integrar al agente como un componente dentro de sistemas mas grandes.

```bash
# Iniciar el servidor ACP
opencode acp

# Con variables de entorno para configurar el modo
OPENCODE_ACP_PORT=4097 opencode acp --hostname 0.0.0.0
```

El protocolo ACP define mensajes con una estructura JSON que incluye tipo (`request`, `response`, `event`), un identificador unico para correlacion peticion-respuesta, el payload con los datos del mensaje, y metadatos opcionales. El flujo tipico comienza con un mensaje `handshake` donde el cliente negocia las capacidades con OpenCode, seguido de mensajes `prompt` que contienen las instrucciones a ejecutar. OpenCode procesa el prompt exactamente igual que si se hubiera invocado desde el TUI, pero devuelve los resultados como eventos JSON estructurados en lugar de texto formateado.

Los casos de uso de ACP incluyen la construccion de interfaces de usuario personalizadas (un dashboard web que envia prompts a OpenCode y muestra los resultados en tiempo real), la integracion en pipelines de procesamiento donde OpenCode actua como un paso mas dentro de un flujo orquestado por herramientas como Apache Airflow o n8n, y el encadenamiento de multiples agentes donde un agente principal delega subtareas a instancias especializadas de OpenCode.

## opencode debug

El comando `debug` agrupa herramientas de diagnostico que exponen el estado interno de OpenCode para identificacion de problemas, verificacion de configuracion y trazado de ejecucion. Es el recurso principal cuando el comportamiento del agente no coincide con lo esperado, ya sea por configuracion incorrecta, conflictos entre plugins o problemas de conectividad con los proveedores de IA.

El subcomando mas utilizado es `debug config`, que muestra la configuracion resuelta tras aplicar todas las capas de precedencia. OpenCode fusiona la configuracion desde multiples fuentes en este orden (de menor a mayor prioridad): valores por defecto del programa, archivo global de configuracion, archivo local del proyecto (`opencode.json`), y variables de entorno. El resultado es la configuracion efectiva que OpenCode utiliza en la sesion actual:

```bash
# Mostrar la configuracion completa resuelta
opencode debug config

# Ver solo un segmento especifico
opencode debug config --providers

# Guardar la configuracion resuelta para analisis
opencode debug config --format json > configuracion_resuelta.json
```

La salida incluye todos los proveedores configurados con sus modelos asociados, las claves API (parcialmente ofuscadas por seguridad), los servidores MCP activos, los plugins cargados, las reglas de permisos, y los valores de tiempo de ejecucion como el directorio de datos o el puerto del servidor. Cuando un valor no es el esperado, `debug config` permite rastrear en que capa se definio y por que prevalecio sobre las demas.

OpenCode ofrece adicionalmente capacidades de diagnostico en tiempo de ejecucion. El flag global `--print-logs` combinado con `--log-level DEBUG` activa la traza completa de ejecucion, que incluye cada llamada a herramienta con sus argumentos y resultados, los mensajes intercambiados con el proveedor (incluyendo el prompt completo y la respuesta raw), las decisiones del gestor de contexto sobre compactacion y resumen, y los tiempos de cada operacion. Esta traza es esencial para depurar problemas de rendimiento, identificar llamadas fallidas a herramientas, o verificar que el modelo esta recibiendo el contexto correcto.

```bash
# Traza completa de ejecucion
opencode run "diagnostica este proyecto" --print-logs --log-level DEBUG 2>traza.log

# Verificar conectividad con el proveedor
opencode debug config --providers --show-keys-status
```

Para problemas de red, la inspeccion del estado de los proveedores en `debug config` muestra si las conexiones estan activas, si los tokens son validos y cuales son los endpoints que se estan utilizando. Esto permite diagnosticar rapidamente fallos de autenticacion, timeouts por configuracion incorrecta de proxies, o bloqueos de red que impidan a OpenCode alcanzar los servidores de los proveedores de IA.

> La informacion mostrada por `debug config` incluye claves API parcialmente visibles. En entornos compartidos, se recomienda redirigir la salida a un archivo y limpiar los valores sensibles antes de compartirla con otros miembros del equipo.

