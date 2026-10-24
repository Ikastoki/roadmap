# OpenCode CLI

## Plan mode versus Build mode

OpenCode CLI distingue dos modos de operacion fundamentales que determinan que acciones puede realizar el agente en cada momento. **Plan mode** es un modo de solo lectura: el agente puede inspeccionar el codigo fuente, navegar por la estructura del proyecto, leer archivos, ejecutar busquedas y proponer cambios, pero nunca escribir archivos ni ejecutar comandos en el sistema. **Build mode**, por el contrario, otorga acceso completo a todas las herramientas, incluyendo escritura de archivos, edicion, ejecucion de comandos bash, creacion de directorios y cualquier operacion que modifique el estado del proyecto o del sistema.

La transicion entre modos se realiza mediante las menciones `@plan` y `@build` en el prompt. Cuando se invoca `@plan`, el agente cambia automaticamente a modo lectura y ofrece un analisis estructurado del problema. Al invocar `@build`, retorna a modo ejecucion. El flujo de trabajo recomendado consiste en planificar primero: analizar el problema, disenar la solucion, identificar los archivos afectados y documentar las decisiones arquitectonicas. Una vez que el plan esta completo y revisado, se cambia a Build mode para que el agente ejecute cada paso de forma precisa. Este enfoque evita modificaciones prematuras, reduce errores y produce un registro claro de que se hizo y por que.

> Plan mode no es un modo limitado, sino un modo de diseno. Las especificaciones que produce son directamente ejecutables por Build mode, lo que convierte la sesion en un ciclo de analisis-implementacion controlado.

## Sesiones efectivas

La calidad de una sesion con OpenCode depende directamente de la claridad con la que se expresan el contexto y los objetivos. Un prompt efectivo incluye el problema concreto, el resultado esperado, las restricciones conocidas y, cuando sea relevante, el enfoque que se quiere seguir. El lenguaje de menciones de OpenCode permite enriquecer el contexto de forma precisa: usar `@` seguido de una ruta de archivo proporciona el contenido completo de ese archivo al agente, lo que evita descripciones vagas y asegura que el agente trabaja sobre el codigo real. De forma similar, el operador `!` permite ejecutar comandos inline y capturar su salida dentro del mismo flujo de conversacion, util para obtener resultados de compilacion, tests o diagnostico sin salir del contexto.

La capacidad de adjuntar archivos extiende aun mas las posibilidades: capturas de pantalla de interfaces de usuario, diagramas de arquitectura en formato imagen, o documentos PDF y hojas de calculo con datos de configuracion o requisitos. Cada uno de estos elementos incrementa el tamano del contexto, lo que tiene implicaciones directas en el coste por consulta (los modelos de lenguaje cobran por token procesado) y en el tiempo de respuesta. Un contexto excesivamente grande puede ralentizar al agente o provocar compactaciones prematuras. La disciplina de contexto es, por tanto, una habilidad clave: incluir solo la informacion necesaria y limpiar el historial cuando sea pertinente.

## Gestion de contexto

El contexto de la conversacion es el estado vivo de la sesion. OpenCode gestiona automaticamente el espacio disponible mediante un sistema de **compactacion**: cuando el contexto se aproxima a los limites de la ventana del modelo, el agente resume los intercambios anteriores para preservar la informacion esencial mientras libera espacio. Este proceso es transparente para el usuario, pero conviene conocerlo porque una compactacion elimina el detalle exacto de conversaciones pasadas. Si se necesita conservar informacion critica, puede hacerse de forma explicita con el comando `/compact`, que fuerza la compactacion en el momento deseado.

Para tareas paralelas que no deben contaminar el contexto principal, OpenCode permite lanzar **subagentes**: sesiones hijas que operan de forma independiente y devuelven unicamente su resultado final. Esto es especialmente util para tareas como analisis de seguridad, revision de codigo en segundo plano o busquedas exhaustivas. El **arbol de sesiones** completa la saga de gestion de contexto: desde cualquier punto de la conversacion se puede bifurcar una sesion hija, explorar una linea alternativa y volver al punto original. La navegacion se realiza con las teclas de flecha en la interfaz TUI, lo que permite moverse entre sesiones hermanas sin perder el estado de ninguna.

> El sistema de contabilidad de tokens de OpenCode expone metricas en tiempo real: tokens de entrada, de salida, total acumulado y estimacion de coste. Monitorizar estos valores ayuda a tomar decisiones informadas sobre cuando compactar o iniciar una nueva sesion.

## Seguridad

OpenCode CLI dispone de un modelo de permisos granular que permite al usuario controlar que acciones puede realizar el agente sin supervision. En el nivel de permisos `ask`, la interfaz TUI muestra cada comando propuesto con un dialogo de aceptacion o denegacion antes de su ejecucion. Esto incluye comandos bash, escritura de archivos y operaciones de red. El usuario puede revisar el comando exacto, el contexto en el que se ejecuta y decidir si permitirlo, denegarlo o recordar la decision para futuras ocasiones.

Para entornos remotos o servidores compartidos, OpenCode soporta la variable de entorno `OPENCODE_SERVER_PASSWORD`, que protege el acceso al modo servidor mediante autenticacion por contrasena. Los datos locales de la aplicacion se almacenan en `~/.local/share/opencode/`, incluyendo configuracion, historial de sesiones, claves de API y cache. La proteccion de este directorio es responsabilidad del usuario, especialmente en entornos compartidos. El poder de OpenCode es considerable: acceso completo al sistema de archivos, ejecucion arbitraria de comandos y capacidad de modificar cualquier recurso. Comprender el alcance de cada permiso que se concede es la primera linea de defensa.

## Compartir sesiones

El comando `/share` genera un enlace en el dominio `opncd.ai` que permite compartir una sesion completa en modo solo lectura. El destinatario puede navegar por el historial de la conversacion, ver los comandos ejecutados y los archivos modificados, pero no interactuar ni modificar nada. El comando `/unshare` revoca el enlace de forma inmediata. La configuracion de comparticion ofrece tres modos: `manual` (solo se comparte cuando se invoca `/share` explicitamente), `auto` (cada sesion genera un enlace automaticamente) y `disabled` (se desactiva toda funcionalidad de comparticion).

El comando `/export` complementa la comparticion generando un archivo Markdown limpio con el contenido de la sesion, util para documentar decisiones tecnicas, adjuntar analisis a informes de incidencias o incorporar el resultado de una investigacion a la documentacion del proyecto. Los casos de uso tipicos incluyen compartir sesiones de depuracion con colegas que necesitan ver exactamente que comandos se ejecutaron y que resultados se obtuvieron, generar documentacion a partir de un analisis exploratorio, o adjuntar una traza completa a un reporte de error en un sistema de ticketing.

