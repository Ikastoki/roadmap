# OpenCode CLI

La linea de comandos ha sido durante decadas el entorno natural de los desarrolladores que buscan precision y control sobre sus herramientas. OpenCode se inscribe en esa tradicion, pero llevando la automatizacion asistida por inteligencia artificial a un nivel de madurez poco comun: no se trata de un chat con capacidad de leer archivos, sino de un agente de codigo autonomo que opera directamente sobre el sistema de archivos, ejecuta comandos, orquesta flujos de trabajo complejos y mantiene contexto conversacional a lo largo de sesiones completas. A diferencia de los asistentes integrados en editores de codigo como GitHub Copilot o Cursor, que operan sobre el fragmento visible en el momento, OpenCode trabaja a nivel de proyecto, con acceso completo al arbol de directorios, capacidad de ejecutar procesos del sistema y la facultad de modificar decenas de archivos de forma coordinada. Esto lo convierte en una herramienta adecuada tanto para tareas de refactorizacion masiva como para el diagnostico profundo de incidencias en bases de codigo extensas.

## Que es OpenCode

OpenCode es un agente de codigo de codigo abierto que se ejecuta integramente en la terminal. Su arquitectura lo distingue de los asistentes de codigo convencionales en varios aspectos fundamentales. El primero es el ambito de operacion: mientras que Copilot sugiere lineas en el archivo abierto y Cursor opera sobre el buffer del editor, OpenCode tiene visibilidad completa del proyecto, puede navegar entre directorios, inspeccionar archivos relacionados y construir un modelo mental de la estructura global antes de proponer cualquier cambio. El segundo es la capacidad de ejecucion: OpenCode puede lanzar comandos de compilacion, ejecutar tests, inspeccionar logs, arrancar servidores de desarrollo y reaccionar a la salida de estos procesos para ajustar su siguiente accion. Esto crea un bucle de retroalimentacion en el que el agente no solo escribe codigo, sino que lo prueba, lo depura y lo corrige de forma autonoma.

OpenCode soporta multiples proveedores de modelo de lenguaje, incluyendo Claude, GPT, Gemini y mas de 75 modelos adicionales a traves de proveedores como Anthropic, OpenAI, Google, AWS Bedrock, Azure, Groq, Together AI y Ollama para ejecucion local. Esta independencia del proveedor significa que el usuario puede elegir el modelo que mejor se ajuste a su tarea o presupuesto sin cambiar de herramienta.

La herramienta ofrece cuatro modos de operacion diferenciados. El modo TUI (Terminal User Interface) proporciona una interfaz interactiva dentro de la terminal con paneles, historial y navegacion mediante teclado. El modo CLI permite ejecutar comandos directos pasando el prompt como argumento, util para scripting y automatizacion. El modo Web expone una interfaz grafica accesible desde el navegador, ideal para equipos que prefieren no trabajar en la terminal. El modo servidor headless convierte a OpenCode en un servicio HTTP con una API completa, permitiendo su integracion en pipelines de CI/CD, entornos de desarrollo remotos o herramientas de terceros.

```bash
# Ejemplo: modo CLI directo
opencode "Explain the architecture of this project"

# Modo TUI interactivo
opencode

# Modo web
opencode --mode web

# Modo servidor headless
opencode --mode server --port 8080
```

> OpenCode es software libre bajo licencia Apache 2.0. El codigo fuente, las issues y la documentacion se encuentran en github.com/anomalyco/opencode.

## Conceptos clave

La sesion conversacional es la unidad fundamental de interaccion con OpenCode. Cada sesion mantiene un historial de mensajes entre el usuario y el agente, y sobre ese historial se construye el contexto que el modelo utiliza para generar respuestas coherentes. A diferencia de los chats ephemerales de los asistentes web, las sesiones de OpenCode son persistentes: se almacenan en el sistema de archivos y pueden reanudarse, ramificarse o compartirse. El mecanismo de compaction permite resumir automaticamente los intercambios antiguos para mantener el contexto dentro de los limites de ventana del modelo sin perder informacion esencial.

OpenCode introduce dos modos de operacion conceptualmente distintos: Plan mode y Build mode. En Plan mode, el agente opera en modo solo lectura: puede explorar el codigo, analizar la estructura del proyecto, leer archivos y hacer preguntas aclaratorias, pero no puede modificar ningun archivo ni ejecutar comandos que alteren el sistema. Esta disociacion entre analisis y ejecucion reduce el riesgo de cambios no deseados durante la fase de comprension del problema. En Build mode, el agente recupera todas sus capacidades: escribir archivos, ejecutar comandos, modificar configuraciones y desplegar cambios. La transicion entre modos puede controlarse manualmente o delegarse al criterio del agente segun la configuracion.

```bash
# Iniciar en Plan mode (solo analisis)
opencode --mode plan

# Iniciar directamente en Build mode
opencode --mode build
```

La arquitectura de subagentes permite a Opencode descomponer tareas complejas en trabajos paralelos. Cuando el agente principal identifica que una tarea contiene subtareas independientes (por ejemplo, refactorizar tres modulos que no comparten dependencias), puede lanzar agentes hijo que trabajan simultaneamente. Cada subagente opera con su propio contexto y su propia ventana de archivos, y los resultados se consolidan al finalizar. Este modelo de ejecucion paralela reduce drasticamente los tiempos de espera en tareas que involucran multiples archivos.

El Model Context Protocol (MCP) es un estandar abierto que permite a OpenCode conectarse con herramientas y fuentes de datos externas. A traves de servidores MCP, el agente puede consultar bases de datos, interactuar con APIs de terceros, buscar en documentacion interna o acceder a sistemas de monitorizacion. Cada servidor MCP expone un conjunto de recursos y herramientas que OpenCode puede invocar de forma transparente durante la conversacion.

```json
{
  "mcpServers": {
    "database": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-server.js"],
      "env": {
        "DB_URL": "postgresql://localhost/mydb"
      }
    }
  }
}
```

El archivo `AGENTS.md` en la raiz del proyecto permite definir reglas e instrucciones especificas que OpenCode debe seguir al trabajar en ese repositorio. A diferencia de un prompt fijo, `AGENTS.md` actua como un conjunto de directrices persistentes que el agente carga automaticamente al iniciar una sesion en ese directorio. Esto es especialmente util para establecer convenciones de codigo, restricciones de seguridad, preferencias de framework o cualquier otra politica que deba aplicarse consistentemente a traves de multiples sesiones.

La gestion de sesiones incluye capacidades de deshacer y rehacer cambios a nivel de operacion, asi como la visualizacion del arbol de sesiones. Cada vez que se ramifica una conversacion (por ejemplo, al explorar un enfoque alternativo sin abandonar el actual), OpenCode registra la bifurcacion y permite navegar entre ramas. El comando `opencode --session tree` muestra esta estructura y facilita la reanudacion de cualquier punto anterior.

```bash
# Listar sesiones disponibles
opencode --session list

# Reanudar una sesion anterior
opencode --session resume "session-id"

# Mostrar el arbol de ramificaciones
opencode --session tree
```

> El coste de las llamadas a modelos de pago se controla mediante limites de gasto configurables por sesion, por dia o por mes. OpenCode muestra una estimacion del coste antes de ejecutar cada operacion que requiera llamadas al modelo.

