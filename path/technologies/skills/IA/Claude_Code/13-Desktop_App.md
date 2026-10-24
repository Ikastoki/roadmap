# Claude Code Desktop

La interfaz de linea de comandos ha sido historically el corazon de Claude Code, pero a partir de abril de 2026, Anthropic introdujo la Desktop App como un entorno dedicado para trabajo agente paralelo, disponible para macOS y Windows. A diferencia de lo que podria pensarse, no se trata de un wrapper grafico del terminal, sino de una aplicacion nativa que ejecuta agentes de Claude en sesiones aisladas con su propio sistema de gestion de proyectos, diffs visuales y un modelo de interaccion donde el desarrollador supervisa mientras la IA ejecuta tareas de forma autonoma.

## Caracteristicas

La Desktop App introduce sesiones paralelas con git isolation, lo que significa que cada sesion de trabajo opera en su propio directorio independiente, evitando conflictos entre tareas concurrentes. Si un agente esta realizando una refactorizacion en una rama y otro esta corrigiendo un bug en otra, ambos pueden trabajar simultaneamente sin interferencias, cada uno con su propio estado de repositorio. Cada sesion mantiene su propio contexto, historial y configuracion de herramientas.

El Side Chat (accesible con `Cmd+;` en macOS o `Ctrl+;` en Windows) permite lanzar preguntas rapidas a un agente auxiliar sin interrumpir el agente principal. Mientras el agente principal continua trabajando en una tarea compleja, el desarrollador puede consultar documentacion, preguntar por fragmentos de codigo o resolver dudas puntuales en un panel lateral que no interfiere con el flujo principal de trabajo.

La aplicacion incorpora un visualizador de diffs integrado que muestra los cambios propuestos por el agente antes de aplicarlos, facilitando la revision. El panel de agente ofrece una vista general de multiples tareas en ejecucion, su estado, progreso y resultados parciales, lo que permite al desarrollador tener visibilidad completa del trabajo que realizan los agentes en paralelo.

Los atajos de teclado principales incluyen `Cmd+N` para abrir una nueva sesion, `Cmd+Shift+D` para abrir el panel de diffs, `Ctrl+O` para alternar el modo de visualizacion entre ventanas y `Cmd+/` para mostrar todos los shortcuts disponibles.

> La Desktop App no reemplaza al CLI; ambos coexisten y comparten el mismo ecosistema. La eleccion depende del flujo de trabajo que prefiera el desarrollador en cada momento.

## CLI vs Desktop

La distincion fundamental entre CLI y Desktop App radica en el modelo de interaccion. El CLI esta optimizado para el flujo "yo escribo codigo, la IA me ayuda": el desarrollador mantiene el control directo del teclado y el editor, invocando a Claude para tareas especificas dentro de su flujo tradicional. La Desktop App, en cambio, esta disenada para el modelo "la IA maneja tareas, yo reviso resultados": el desarrollador define objetivos y supervisa el progreso mientras los agentes trabajan de forma autonoma en multiples frentes.

Cuando se trabaja en el CLI y se necesita la potencia de las sesiones paralelas o el entorno visual de la Desktop App, el comando `/desktop` transfiere la sesion actual del CLI a la Desktop App, manteniendo el contexto, el historial y el estado del agente. Esta transicion es fluida porque ambas aplicaciones comparten la misma autenticacion, los mismos datos almacenados en `~/.claude/` y el mismo sistema de archivos de proyecto.

La compatibilidad entre CLI y Desktop es total: las sesiones pueden moverse de un entorno a otro, los historiales de conversacion son accesibles desde ambos y las configuraciones de herramientas y permisos se sincronizan automaticamente. Un desarrollador puede comenzar una tarea en el terminal mientras viaja, continuarla en la Desktop App al llegar a la oficina y revisar los resultados desde cualquiera de los dos entornos.

> La Desktop App no esta disponible en Linux. Los usuarios de Linux continuan usando el CLI como unico punto de acceso, aunque con la misma capacidad de ejecutar agentes y la misma compatibilidad de configuracion y datos.

