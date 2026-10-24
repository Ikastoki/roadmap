# Claude Code CLI

Dentro de una sesion interactiva, escribir `/` despliega automaticamente el menu completo de comandos disponibles. Estos comandos slash constituyen el vocabulario operativo de Claude Code: no son simples atajos, sino instrucciones estructuradas que activan comportamientos especificos del agente, desde la gestion del contexto y la memoria hasta la ejecucion de revisiones de codigo o la coordinacion de subagentes. Cada comando slash expone una capacidad del sistema que, sin el, requeriria multiples intercambios conversacionales o manipulacion manual de archivos de configuracion.

## Gestion de sesion

El comando `/init` crea o actualiza el archivo `CLAUDE.md` en la raiz del proyecto, permitiendo definir instrucciones persistentes que el agente cargara automaticamente en cada nueva sesion. Es el mecanismo fundamental para establecer convenciones, restricciones y contexto permanente del proyecto. `/plan` activa el modo de solo lectura, donde Claude Code analiza el codigo y elabora estrategias sin ejecutar cambios en el sistema de ficheros, ideal para fases de diseno o diagnosis preliminar.

```bash
/plan "disena la arquitectura para el modulo de autenticacion OAuth2"
```

`/compact` resume el historial de la conversacion para liberar tokens de contexto sin perder informacion critica, operacion que se vuelve necesaria en sesiones extensas donde el limite de ventana de contexto se aproxima. `/clear` inicia una conversacion completamente nueva manteniendo la sesion activa, mientras que `/undo` deshace el ultimo cambio realizado y `/rewind` retrocede la conversacion hasta un checkpoint anterior, util cuando una exploracion no ha dado los resultados esperados.

La ramificacion de conversaciones se gestiona con `/branch`, que crea una rama paralela a partir de un punto de la conversacion sin perder el progreso original. `/fork` envia una copia de la conversacion a una sesion en background, permitiendo continuar el trabajo principal mientras el agente procesa tareas secundarias. `/resume` reanuda una sesion suspendida, y `/background` o `/bg` desacopla la sesion actual para ejecutarla en segundo plano, liberando el terminal para otras tareas. `/tasks` lista todos los trabajos background activos y `/subtask` delega una tarea concreta a un subagente especializado.

```bash
/subtask "refactoriza el controlador UserController siguiendo los patrones del proyecto"
```

`/add-dir` incorpora un directorio completo al contexto de trabajo, `/cd` cambia el directorio de trabajo dentro de la sesion sin necesidad de salir del terminal, `/rename` asigna un nombre descriptivo a la sesion para identificarla facilmente entre multiples instancias, `/color` modifica el color de la barra de estado para distinguir visualmente sesiones paralelas y `/btw` permite realizar una pregunta lateral que no se registra en el historial principal de la conversacion.

> `/btw` es especialmente util para consultas instrumentales como "que version de Node.js requiere este proyecto?" sin contaminar el hilo principal de trabajo con informacion auxiliar.

## Revision y calidad

`/code-review` ejecuta una revision tecnica del diff actual respecto al ultimo commit, con niveles de profundidad que van desde `low` hasta `ultra`. El nivel `ultra` realiza un analisis exhaustivo linea por linea, mientras que `low` ofrece una verificacion superficial de problemas evidentes. El flag `--fix` aplica automaticamente las correcciones propuestas y `--comment` publica los resultados como comentarios en el PR correspondiente.

```bash
/code-review ultra --fix
```

`/review` proporciona una revision rapida de un Pull Request completo sin necesidad de que el codigo este presente localmente, analizando el diff remoto y generando observaciones estructuradas. `/security-review` se centra exclusivamente en vulnerabilidades: inyeccion SQL, exposicion de secretos, desbordamiento de buffers, configuraciones de CORS inseguras y otras superficies de ataque comunes. `/simplify` examina el codigo modificado en la sesion y sugiere refactorizaciones que reducen la complejidad ciclomatica mejorando la legibilidad sin alterar la funcionalidad.

`/autofix-pr` automatiza completamente el ciclo de correccion: detecta fallos en CI, lee los comentarios del PR, implementa las correcciones necesarias y actualiza el PR. `/diff` muestra el conjunto de cambios realizados hasta el momento en la sesion y `/status` presenta un resumen del estado actual incluyendo el modelo activo, el esfuerzo de razonamiento configurado y los archivos modificados.

## Configuracion y herramientas

`/config` o `/settings` permite ajustar parametros de configuracion en tiempo de ejecucion mediante pares `clave=valor`. Desde cambiar el modelo (`/config model=claude-sonnet-4-20250624`) hasta ajustar el nivel de detalle de las respuestas, la configuracion es inmediata y no requiere reiniciar la sesion. `/permissions` revisa y modifica los permisos que el agente tiene sobre el sistema de ficheros, estableciendo restricciones granulares sobre que directorios puede leer o modificar.

```bash
/config model=claude-sonnet-4-20250624 theme=catppuccin-mocha effort=high
```

`/hooks` gestiona los hooks del proyecto que se ejecutan en eventos como pre-commit o post-edit, permitiendo integrar Claude Code con flujos de trabajo personalizados. `/mcp` administra los servidores MCP (Model Context Protocol) conectados a la sesion, listando, anadiendo o eliminando conexiones a herramientas externas. `/agents` lista los subagentes disponibles y permite cambiar entre ellos, y `/model` cambia el modelo de lenguaje activo sobre la marcha.

`/effort` controla el nivel de esfuerzo de razonamiento que el modelo dedica a cada respuesta, con valores que van desde `low` (respuestas rapidas) hasta `max` (analisis profundo). `/themes` aplica esquemas visuales a la interfaz del terminal y `/keybindings` configura los atajos de teclado. `/doctor` ejecuta un diagnostico completo del entorno y `/debug` activa la salida detallada de depuracion para identificar problemas en el comportamiento del agente.

```bash
/effort max
```

`/export` descarga la conversacion completa en formato Markdown o JSON, `/copy` copia la ultima respuesta al portapapeles del sistema y `/usage` o `/cost` muestra estadisticas detalladas de consumo de tokens y coste acumulado de la sesion, util para monitorizar el gasto en entornos donde la facturacion es por consumo.

## Trabajo en equipo

`/batch` ejecuta cambios masivos sobre multiples worktrees de Git simultaneamente, permitiendo propagar una modificacion a todas las ramas activas del repositorio. `/deep-research` lanza una investigacion web multietapa donde el agente formula preguntas, busca informacion, sintetiza resultados y profundiza iterativamente hasta alcanzar el nivel de detalle requerido. `/memory` gestiona la memoria persistente del proyecto, almacenando y recuperando informacion entre sesiones a traves del archivo `CLAUDE.md` y la memoria interna del agente.

```bash
/deep-research "analiza las alternativas de autenticacion sin password en 2026"
```

`/design-sync` alinea la implementacion actual con documentos de diseno, detectando desviaciones y sugiriendo correcciones. `/teleport` trae una sesion web al terminal y `/remote-control` permite continuar una sesion desde otro dispositivo. `/team-onboarding` genera una guia de incorporacion para nuevos miembros del equipo basada en la estructura y configuracion del proyecto. `/insights` analiza patrones en el codigo y sugiere mejoras arquitectonicas. `/voice` activa el dictado por voz y `/powerup` ofrece lecciones interactivas para dominar caracteristicas avanzadas de Claude Code. `/reload-skills` recarga las skills definidas en el proyecto sin necesidad de reiniciar la sesion.

> Los comandos de equipo requieren que el proyecto este configurado con un archivo `CLAUDE.md` adecuado y, en algunos casos, integracion con servicios externos como repositorios remotos o APIs de terceros.

La comprension de los comandos slash transforma la interaccion con Claude Code de una conversacion abierta a un dialogo dirigido. Cada comando representa una operacion atómica que el agente ejecuta con precision, eliminando la ambiguedad del lenguaje natural para tareas que requieren comportamiento deterministico. La combinacion de comandos en secuencia permite construir flujos de trabajo complejos sin salir del terminal, convirtiendo a Claude Code en un sistema operativo de desarrollo mas que en un simple asistente conversacional.

