# Claude Code CLI

La linea de comandos ha sido durante decadas el entorno natural de los desarrolladores que buscan precision y control, y con la llegada de los agentes de codigo basados en inteligencia artificial la terminal recupera un protagonismo que parecia cedido a los entornos graficos integrados. Claude Code, desarrollado por Anthropic, representa una de las implementaciones mas maduras de agente de codificacion autonomo: opera directamente sobre el sistema de archivos, lee y modifica codigo, ejecuta comandos del sistema, interactua con Git y mantiene conversaciones contextuales de multiples turnos, todo desde la terminal. A diferencia de las herramientas de autocompletado que operan sobre el fragmento visible del archivo abierto, Claude Code trabaja a nivel de proyecto, con capacidad de comprender la arquitectura completa, navegar entre directorios y coordinar cambios en decenas de archivos de forma autonoma.

## Que es Claude Code

Claude Code es el agente de codificacion oficial de Anthropic, disenado para ejecutarse en la terminal como un asistente de desarrollo autonomo. Su diferencia fundamental respecto a otras herramientas del mercado radica en el ambito de operacion y el nivel de agencia. Mientras que los asistentes tradicionales se limitan a sugerir fragmentos de codigo en el contexto del archivo abierto, Claude Code examina la totalidad del proyecto, construye un modelo mental de su estructura, ejecuta comandos de compilacion y prueba, y modifica archivos de forma coordinada para lograr un objetivo expresado en lenguaje natural.

El motor subyacente son los modelos Claude de Anthropic, disponibles en tres variantes principales. Opus es el modelo mas capaz, adecuado para tareas complejas que requieren razonamiento profundo y analisis exhaustivo. Sonnet ofrece el mejor equilibrio entre capacidad y velocidad, siendo la opcion predeterminada para la mayoria de las tareas cotidianas. Haiku es el modelo mas rapido y ligero, optimizado para tareas simples que requieren respuestas inmediatas. La seleccion del modelo puede realizarse por sesion o por tarea, permitiendo al usuario ajustar el equilibrio entre coste, velocidad y calidad segun sus necesidades.

Claude Code puede utilizarse a traves de tres interfaces diferenciadas. La CLI es el modo nativo, accesible directamente desde cualquier terminal. La aplicacion de escritorio esta disponible para macOS y Windows, proporcionando una experiencia integrada con el sistema operativo. Las extensiones para IDE permiten incrustar el agente directamente en el flujo de trabajo del editor, combinando la potencia del analisis a nivel de proyecto con la comodidad del entorno de desarrollo integrado.

```bash
# Iniciar sesion interactiva en la terminal
claude

# Ejecutar un comando directo
claude "Refactoriza este modulo para usar async/await"

# Usar con un modelo especifico
claude --model opus

# Iniciar en modo plan (solo lectura)
claude --mode plan
```

> Claude Code requiere una clave de API de Anthropic para funcionar. El coste de las llamadas se descuenta del credito disponible en la cuenta, y la herramienta muestra una estimacion del gasto antes de ejecutar operaciones que requieran multiples llamadas al modelo.

## Conceptos clave

La sesion conversacional es la unidad fundamental de interaccion con Claude Code. Cada sesion mantiene un historial de mensajes entre el usuario y el agente, y sobre ese historial se construye el contexto que el modelo utiliza para generar respuestas coherentes. A diferencia de los chats efimeros de los asistentes web, las sesiones de Claude Code son persistentes: se almacenan localmente y pueden reanudarse, ramificarse o compartirse. El sistema implementa compresion automatica del contexto, resumiendo intercambios antiguos para mantener la conversacion dentro de los limites de la ventana del modelo sin perder informacion relevante.

Claude Code introduce dos modos de operacion conceptualmente distintos: Plan mode y Build mode. En Plan mode, el agente opera en modo exclusivamente analitico: puede explorar el codigo, leer archivos, analizar la estructura del proyecto y formular preguntas al usuario, pero no puede modificar ningun archivo ni ejecutar comandos que alteren el sistema. Esta separacion entre analisis y ejecucion reduce drasticamente el riesgo de cambios no deseados durante la fase de comprension del problema. En Build mode, el agente recupera todas sus capacidades operativas: escribir archivos, ejecutar comandos, realizar commits de Git y modificar configuraciones. La transicion entre modos puede controlarse manualmente o delegarse al criterio del agente segun la configuracion de permisos establecida.

```bash
# Iniciar sesion en modo plan
claude --mode plan

# Ejecutar un analisis sin riesgo de modificaciones
claude --mode plan "Cual es la cobertura de tests actual?"
```

La arquitectura de subagentes permite a Claude Code descomponer tareas complejas en trabajos paralelos. Cuando el agente principal identifica que una tarea contiene subtareas independientes (por ejemplo, refactorizar tres modulos que no comparten dependencias), puede lanzar agentes hijo que trabajan simultaneamente. Cada subagente opera con su propio contexto y su propia ventana de archivos, y los resultados se consolidan al finalizar la ejecucion paralela. Este modelo reduce significativamente los tiempos de espera en tareas que involucran multiples archivos y permite abordar proyectos de gran envergadura de forma eficiente.

El Model Context Protocol (MCP) es un estandar abierto disenado por Anthropic que permite a Claude Code conectarse con herramientas y fuentes de datos externas. A traves de servidores MCP, el agente puede consultar bases de datos, interactuar con APIs de terceros, buscar en documentacion interna o acceder a sistemas de monitorizacion. Cada servidor MCP expone un conjunto de recursos y herramientas que Claude Code puede invocar de forma transparente durante la conversacion. MCP sigue el mismo protocolo que utilizan otros clientes de Anthropic, lo que permite reutilizar servidores MCP existentes en la comunidad.

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

El archivo `CLAUDE.md` en la raiz del proyecto actua como un mecanismo de memoria persistente y definicion de reglas. A diferencia de un prompt fijo, este archivo contiene directrices que Claude Code carga automaticamente al iniciar una sesion en ese directorio. Es el lugar adecuado para establecer convenciones de codificacion, restricciones de seguridad, preferencias de frameworks, descripciones de la arquitectura del proyecto o cualquier otra politica que deba aplicarse consistentemente a traves de multiples sesiones y miembros del equipo.

Claude Code ofrece tres modos de permisos que determinan el nivel de autonomia del agente. El modo plan restringe al agente a operaciones de solo lectura. El modo auto permite al agente ejecutar comandos y modificar archivos despues de mostrar un resumen de los cambios propuestos. El modo bypass concede al agente capacidad de ejecucion sin confirmacion previa, util para tareas automatizadas en entornos de confianza.

```bash
# Modo auto: muestra resumen antes de ejecutar
claude --permission auto

# Modo bypass: ejecucion sin confirmacion
claude --permission bypass "Corrige todos los errores de lint en src/"
```

La gestion de sesiones incluye capacidades avanzadas como checkpoints, ramificacion y ejecucion en segundo plano. Los checkpoints permiten guardar el estado de la sesion en un punto concreto, facilitando la exploracion de alternativas sin riesgo. Las ramas (branches) permiten bifurcar la conversacion para explorar enfoques diferentes sin abandonar la linea principal. La ejecucion en segundo plano (background sessions) permite a Claude Code continuar trabajando mientras el usuario realiza otras tareas. Los Dynamic Workflows extienden esta capacidad permitiendo definir flujos de trabajo paralelos a gran escala que Claude Code ejecuta de forma coordinada, ideal para refactorizaciones masivas o migraciones de codigo que afectan a cientos de archivos.

```bash
# Crear un checkpoint en la sesion actual
claude --checkpoint save "antes-de-refactor"

# Reanudar desde un checkpoint
claude --checkpoint load "antes-de-refactor"

# Iniciar sesion en segundo plano
claude --background "Migra todos los styled-components a Emotion"
```

> Dynamic Workflows es una funcionalidad avanzada que permite orquestar cambios paralelos a gran escala. Al combinarla con subagentes, Claude Code puede abordar tareas como la migracion de un sistema de estilos a otro, la actualizacion de una API en decenas de controladores o la normalizacion de imports en todo el proyecto de forma coordinada y eficiente.

