# OpenCode CLI

La interfaz TUI (Terminal User Interface) de OpenCode convierte la terminal en un entorno de desarrollo asistido completo, con paneles divididos, navegacion por teclado, autocompletado contextual y gestion de sesiones visual. A diferencia del modo CLI puro, donde se pasa un prompt como argumento y se obtiene una respuesta inmediata, el modo TUI mantiene una sesion conversacional persistente en la que el usuario y el agente interactuan de forma iterativa, consultando el historial, modificando archivos y revisando diferencias en tiempo real. Esta interfaz esta construida sobre el framework Ratatui, lo que garantiza compatibilidad con cualquier emulador de terminal moderno y un rendimiento fluido incluso en conexiones SSH lentas.

## Interfaz de chat

El componente central del TUI es el panel de conversacion, que ocupa la mayor parte de la pantalla y muestra el historial de mensajes entre el usuario y el agente. En la parte inferior se encuentra el area de entrada de prompt, un campo de texto con soporte para multilinea donde se redactan las instrucciones. Al comenzar a escribir, OpenCode activa la busqueda difusa de referencias a archivos mediante el prefijo `@`: al escribir `@` seguido de un patron, aparece una lista de archivos del proyecto que coinciden, permitiendo seleccionar y adjuntar referencias sin necesidad de escribir rutas completas. Este mecanismo usa el arbol de directorios indexado en memoria y soporta coincidencias parciales en nombres y extensiones.

```bash
# Ejemplos de referencia a archivos con @
@utils/helpers.py          # coincide con la ruta exacta
@logger                    # cualquier archivo que contenga "logger"
@*.test.ts                 # todos los test de TypeScript
```

El prefijo `!` permite ejecutar comandos de bash directamente desde el prompt sin salir del TUI. Cuando una linea comienza con `!`, OpenCode interpreta el resto como un comando del sistema, lo ejecuta y muestra la salida en el panel de conversacion. Esto es util para tareas rapidas como inspeccionar procesos, lanzar builds o consultar el estado del repositorio sin cambiar de ventana.

```bash
!git status
!npm run build
!lsof -i :3000
```

El TUI organiza la informacion en un layout de paneles divididos. El panel principal muestra la conversacion; un panel secundario, que puede mostrarse u ocultarse con un atajo de teclado, presenta los diffs de los archivos modificados durante la sesion. Cuando OpenCode aplica cambios, estos se reflejan inmediatamente en el panel de diff, permitiendo revisar cada modificacion antes de aceptarla. La navegacion entre paneles se realiza con las teclas de tabulacion o las flechas de direccion, y el autocompletado contextual sugiere comandos slash, nombres de archivo y rutas validas a medida que se escribe.

## Comandos slash (/)

Los comandos slash son instrucciones prefijadas con `/` que se introducen en el prompt para ejecutar operaciones especificas sobre el estado de la sesion o la configuracion de OpenCode. Cada comando actua como una llamada a funcion interna que modifica el comportamiento del agente o la interfaz sin necesidad de redactar un prompt explicativo.

`/help` muestra la lista completa de comandos disponibles con una breve descripcion de cada uno. Es el punto de entrada para usuarios que no recuerdan la sintaxis exacta de un comando especifico.

`/init` crea o actualiza el archivo `AGENTS.md` en la raiz del proyecto. Si el archivo no existe, OpenCode lo genera con una plantilla basica que el usuario puede personalizar. Si ya existe, el comando abre un editor para modificar su contenido. Este archivo actua como conjunto de instrucciones persistentes que el agente carga automaticamente al iniciar sesion en ese directorio.

`/connect` inicia un asistente interactivo para anadir o configurar un proveedor de modelo. El comando guia al usuario a traves de la seleccion del proveedor, la introduccion de la clave API y la configuracion de parametros como el modelo por defecto y los limites de peticiones. Es la alternativa interactiva a la edicion manual de `opencode.json`.

`/models` lista los modelos disponibles para el proveedor activo y permite seleccionar cual se usara en la sesion actual. La seleccion persiste durante la sesion pero no modifica la configuracion global.

`/new` inicia una nueva sesion vacia, cerrando la actual. OpenCode guarda automaticamente la sesion anterior antes de crear la nueva, por lo que no se pierde el historial. Es equivalente a `opencode --session new`.

`/sessions` lista todas las sesiones almacenadas y permite cambiar a cualquiera de ellas mediante seleccion interactiva. Las sesiones se identifican por un nombre descriptivo y una marca de tiempo. Este comando es la puerta de entrada a la navegacion entre sesiones sin salir del TUI.

`/undo` revierte el ultimo mensaje del agente y todos los cambios de archivo asociados. OpenCode mantiene un historial de operaciones que permite deshacer no solo el ultimo intercambio, sino cualquier accion previa siempre que no haya sido purgada por una compactacion. El comando `/redo` restaura la accion que se acaba de deshacer.

`/compact` activa manualmente el mecanismo de compactacion de contexto. OpenCode resume automaticamente los intercambios mas antiguos de la conversacion cuando se acerca al limite de tokens del modelo, pero este comando fuerza la compactacion inmediatamente. El proceso selecciona los segmentos de la conversacion con menor relevancia para la tarea actual, los sintetiza en un resumen breve y los elimina del contexto activo, manteniendo intactos los archivos referenciados y las decisiones de diseno importantes.

`/share` genera un enlace compartible de la sesion actual, y `/unshare` lo revoca. La sesion compartida es accesible en modo solo lectura a traves de la interfaz web de OpenCode.

`/export` convierte la sesion actual a un archivo Markdown, preservando el formato de los bloques de codigo, la estructura de la conversacion y los diffs. Es util para documentar decisiones tecnicas o compartir sesiones con personas que no tienen acceso a OpenCode.

`/editor` abre el prompt actual en el editor de texto externo configurado en la variable de entorno `$EDITOR`. Al guardar y cerrar el editor, el contenido se transfiere al campo de prompt del TUI. Esto permite redactar instrucciones extensas con todas las capacidades de un editor de texto completo.

`/details` alterna la visualizacion de informacion detallada de ejecucion: tokens utilizados, coste estimado, archivos modificados y tiempo de respuesta del modelo. `/thinking` controla la visibilidad de los bloques de razonamiento intermedio del modelo, cuando el proveedor los soporta.

`/themes` abre el selector de temas visuales. OpenCode incluye temas claro y oscuro, ademas de esquemas de color personalizables mediante el archivo `tui.json`.

`/exit` y `/quit` cierran la sesion actual y finalizan el proceso de OpenCode. Ambos comandos son equivalentes y guardan automaticamente el estado de la sesion antes de salir.

```bash
# Uso tipico de comandos slash en el TUI
/connect                    # anadir un proveedor
/models                     # seleccionar modelo
/compact                    # forzar compactacion de contexto
/export session-final.md    # exportar a archivo Markdown
```

> El mecanismo de compactacion no elimina informacion de forma permanente. Los segmentos resumidos se conservan en el archivo de sesion en disco y pueden restaurarse si se revisita esa parte de la conversacion. La compactacion solo afecta al contexto activo que se envia al modelo en cada peticion.

## Atajos de teclado

OpenCode implementa un sistema de atajos basado en una tecla lider (leader key). Por defecto, la tecla lider es `Ctrl+X`. Tras pulsarla, el TUI espera una segunda pulsacion que determina la accion a ejecutar. Este esquema evita conflictos con los atajos nativos del emulador de terminal y permite un numero elevado de combinaciones sin saturar la memoria muscular del usuario.

La combinacion `Ctrl+X U` ejecuta undo sobre el ultimo mensaje, mientras que `Ctrl+X R` ejecuta redo. `Ctrl+X N` inicia una nueva sesion y `Ctrl+X L` muestra la lista de sesiones para seleccionar y reanudar. `Ctrl+X C` fuerza la compactacion manual del contexto. `Ctrl+X M` abre el selector de modelos y `Ctrl+X T` el selector de temas. `Ctrl+X Q` cierra la sesion y sale de OpenCode. `Ctrl+X E` abre el prompt en el editor externo y `Ctrl+X X` exporta la sesion a Markdown.

Ademas del sistema de tecla lider, `Ctrl+P` abre la paleta de comandos, un menu de busqueda textual que permite localizar y ejecutar cualquier comando disponible sin recordar su atajo exacto. `Tab` cicla entre los agentes primarios cuando se trabaja con multiples agentes en paralelo. `Ctrl+T` cicla entre las variantes de modelo disponibles para el proveedor activo (por ejemplo, entre Claude Sonnet y Haiku, o entre GPT-4 y GPT-4-mini). Las teclas de flecha arriba y abajo permiten navegar por el historial de promps de la sesion actual, mientras que en el arbol de sesiones hijas (child sessions), las flechas izquierda y derecha navegan entre las ramas de la conversacion.

```bash
# Resumen de combinaciones frecuentes
Ctrl+X U    # Undo
Ctrl+X R    # Redo
Ctrl+X N    # Nueva sesion
Ctrl+X L    # Listar sesiones
Ctrl+P      # Paleta de comandos
Tab         # Ciclar agente primario
Ctrl+T      # Ciclar variante de modelo
```

## Configuracion TUI

Los aspectos visuales y de comportamiento de la interfaz TUI se configuran a traves del archivo `tui.json`, ubicado en el directorio de configuracion de OpenCode (`~/.config/opencode/tui.json`). Este archivo es independiente de `opencode.json` y se centra exclusivamente en la experiencia de usuario de la terminal.

El campo `theme` define el esquema de colores visual. Puede ser `"light"`, `"dark"` o una referencia a un tema personalizado. OpenCode permite definir temas completos especificando colores para cada elemento de la interfaz: fondo, texto, paneles, bordes, selecciones, enlaces y bloques de codigo. Los temas personalizados se definen en el mismo archivo `tui.json` bajo la clave `themes`.

El campo `keybinds` permite redefinir cualquier atajo de teclado. Cada entrada asigna una combinacion de teclas a una accion de OpenCode, permitiendo adaptar el sistema de navegacion a las preferencias del usuario o a la disposicion del teclado. Las combinaciones pueden incluir modificadores como `ctrl`, `alt`, `shift` y `super`.

`leader_timeout` controla el tiempo maximo, en milisegundos, que el TUI espera tras pulsar la tecla lider antes de cancelar la combinacion. Un valor de `500` significa que el usuario tiene medio segundo para pulsar la segunda tecla. `scroll_speed` define la velocidad de desplazamiento en lineas por pulsacion, y `scroll_acceleration` activa la aceleracion progresiva: cuanto mas tiempo se mantenga pulsada la tecla de desplazamiento, mas rapido se mueve el contenido.

`diff_style` determina como se muestran las diferencias de los archivos modificados. El valor `"auto"` presenta los diffs en linea, integrados en el panel de conversacion, mientras que `"stacked"` los muestra en un panel separado apilado debajo de la conversacion principal.

`mouse` habilita o deshabilita la captura de eventos del raton. Cuando esta activo (`true`), se puede hacer clic para seleccionar paneles, scroll para navegar y clic en enlaces para abrirlos. `attention` controla las notificaciones visuales y sonoras: cuando esta en `"all"`, OpenCode notifica al usuario cuando el agente completa una tarea o cuando solicita intervencion; en `"minimal"`, solo notifica las solicitudes de intervencion; en `"off"`, suprime todas las notificaciones.

```json
{
  "theme": "dark",
  "keybinds": {
    "ctrl+y u": "undo",
    "ctrl+y r": "redo"
  },
  "leader_timeout": 600,
  "scroll_speed": 3,
  "scroll_acceleration": true,
  "diff_style": "auto",
  "mouse": true,
  "attention": "minimal"
}
```

> Los cambios en `tui.json` se aplican en caliente: no es necesario reiniciar OpenCode para que las modificaciones surtan efecto. Basta con guardar el archivo y el TUI lo relee automaticamente en la siguiente interaccion.

