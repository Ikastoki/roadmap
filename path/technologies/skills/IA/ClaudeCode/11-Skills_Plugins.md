# Claude Code CLI

## Skills (SKILL.md)

Una skill es un conjunto de instrucciones reutilizables que se activa bajo demanda mediante un comando slash, no un conjunto de reglas permanentes que envenenan el contexto de toda la sesion. Mientras que `CLAUDE.md` contiene directrices omnipresentes que el agente debe considerar en cada interaccion, las skills representan modulos opt-in: el usuario invoca explícitamente una skill cuando la tarea lo requiere, y el agente carga las instrucciones asociadas unicamente durante la ejecucion de ese comando. Esta distincion es fundamental para la higiene del contexto: las skills evitan que instrucciones especializadas contaminen el prompt base y permiten construir una biblioteca de capacidades reutilizables sin saturar la ventana de atencion del modelo.

Cada skill reside en su propio directorio dentro de `.claude/skills/<nombre>/` y se define mediante un archivo `SKILL.md` con frontmatter YAML. El frontmatter declara metadatos esenciales: `name` (identificador unico del comando slash), `description` (texto que aparece en la ayuda y el autocompletado), `model` (modelo especifico opcional para ejecutar la skill) y `temperature` (control de creatividad, tipicamente entre 0.0 y 1.0). El cuerpo del archivo contiene las instrucciones en Markdown que el agente seguira al invocarse la skill.

```yaml
---
name: code-review
description: Realiza una revision de codigo exhaustiva del directorio especificado
model: claude-opus-4-20260514
temperature: 0.2
---
```

Claude Code incluye un conjunto de skills bundled listas para usar sin configuracion adicional. `/code-review` ejecuta una auditoria tecnica del codigo en el directorio actual o en la ruta que se le pase como argumento, analizando calidad, seguridad y rendimiento. `/batch` permite enviar multiples solicitudes en paralelo desde un archivo JSON, optimizando el rendimiento en tareas repetitivas como la migracion de codigo o la generacion de documentacion. `/deep-research` lanza un proceso de investigacion autonomo que navega por la base de codigo, consulta documentacion y sintetiza hallazgos en un informe estructurado. `/claude-api` expone las capacidades de la API de Claude directamente desde el terminal para prototipado rapido. `/dataviz` genera visualizaciones de datos analizando archivos CSV o JSON y produciendo graficos con librerias como D3 o Matplotlib. `/debug` inicia una sesion diagnostica que ejecuta el codigo, inspecciona errores y propone correcciones. `/design-sync` revisa la consistencia entre diseno y codigo comparando componentes con especificaciones de Figma o Storybook. `/simplify` refactoriza codigo complejo reduciendo la complejidad ciclomatica y mejorando la legibilidad.

```bash
/code-review src/                 # Revisa el directorio src/
/batch solicitudes.json           # Ejecuta solicitudes por lotes
/deep-research "migrar a React 19" # Investigacion exhaustiva
/simplify src/utils/complejo.js   # Simplifica codigo
```

El comando `/reload-skills` recarga todas las skills desde el sistema de ficheros sin necesidad de reiniciar Claude Code. Esto es util durante el desarrollo de skills personalizadas, donde el ciclo de edicion-prueba se acelera al evitar el reinicio del agente. El comando escanea recursivamente `.claude/skills/`, valida el frontmatter YAML de cada `SKILL.md` y registra los comandos slash correspondientes, notificando errores de sintaxis o conflictos de nombres.

> El directorio `.claude/skills/` puede incluirse en el repositorio del proyecto, permitiendo que las skills personalizadas se distribuyan automaticamente a todo el equipo mediante el control de versiones. Cada skill puede contener ademas archivos auxiliares (templates, scripts, datasets) referenciados desde el `SKILL.md`.

La diferencia con `CLAUDE.md` es estrategica: mientras que `CLAUDE.md` establece reglas persistentes que el agente debe acatar en todo momento -- convenciones de codigo, arquitectura del proyecto, restricciones de seguridad --, las skills son invocaciones puntuales que aportan contexto especializado solo cuando se necesitan. Un proyecto puede tener un `CLAUDE.md` que defina el stack tecnologico y las normas de estilo, y al mismo tiempo disponer de skills para auditoria de seguridad, generacion de informes o migracion entre frameworks. Esta separacion mantiene el contexto ligero y las capacidades modulares.

```yaml
---
name: audit-seguridad
description: Escanea dependencias y configura en busca de vulnerabilidades
model: claude-opus-4-20260514
temperature: 0.1
---

Revisa el archivo `package.json` y busca dependencias con vulnerabilidades conocidas.

1. Ejecuta `npm audit --json` y analiza el resultado.
2. Examina la configuracion de seguridad en `.env.example` y `settings.json`.
3. Identifica permisos excesivos en archivos de despliegue (Dockerfile, CI/CD).
4. Genera un informe con severidad, paquete afectado y version recomendada.
```

## Plugins

El sistema de plugins de Claude Code extiende las capacidades del agente mas alla de sus herramientas nativas mediante modulos instalables desde el ecosistema npm. Mientras que las skills son instrucciones que guian el comportamiento del agente, los plugins son paquetes de software que pueden registrar nuevas herramientas, monitores que se ejecutan al inicio de la sesion y comandos personalizados que se integran en el flujo de trabajo del terminal. La instalacion se realiza a traves del comando `claude plugin install <plugin>`, que descarga el paquete desde el registro npm y lo configura automaticamente en el directorio `.claude/plugins/` del proyecto.

Los plugins oficiales de Anthropic utilizan el prefijo `@claude-plugins-official` y ofrecen integraciones validadas con herramientas del ecosistema de desarrollo. Entre ellos se encuentran conectores para sistemas de CI/CD, analizadores estaticos de codigo, integraciones con gestores de incidencias (Jira, Linear, GitHub Issues) y adaptadores para frameworks de testing. La curacion oficial garantiza compatibilidad con la version actual del CLI y adhesión a las politicas de seguridad del agente.

```bash
claude plugin install @claude-plugins-official/eslint
claude plugin install @claude-plugins-official/jira-linear
claude plugin install mi-plugin-personalizado  # Plugin npm generico
```

Cada plugin instalado se registra en `.claude/plugins/<nombre>/` y puede ser configurado desde `settings.json` en la seccion `plugins`, donde se controla su habilitacion y parametros especificos. El soporte para plugins locales -- aquellos que residen en el sistema de ficheros sin publicarse en npm -- se logra copiandolos directamente en `.claude/plugins/` y anadiendo la entrada correspondiente en la configuracion, lo que permite desarrollar y probar extensiones privadas antes de su publicacion.

La arquitectura de plugins expone dos mecanismos de extension principales. Las **custom tools** permiten que un plugin registre herramientas nuevas que el agente puede invocar durante la sesion, ampliando su repertorio operativo con capacidades como interactuar con APIs externas, manipular formatos de datos especificos o ejecutar analizadores especializados. Los **monitors** son scripts que se ejecutan automaticamente al inicio de cada sesion para verificar el estado del entorno: comprobar la version del runtime, validar la presencia de dependencias criticas o alertar sobre configuraciones obsoletas. Este modelo convierte a Claude Code en una plataforma extensible donde cada equipo puede moldear el agente a su flujo de trabajo sin esperar a que las funcionalidades lleguen al nucleo del producto.

> Los plugins que registran monitores deben ser eficientes: un monitor que tarde mas de un par de segundos en ejecutarse retrasa el inicio de la sesion. Para operaciones pesadas, el diseno recomendado es delegar la carga a una custom tool que se invoque bajo demanda.

El ecosistema de plugins se encuentra en crecimiento activo, con contribuciones tanto de Anthropic como de la comunidad open-source. La combinacion de skills como modulos de instruccion ligera y plugins como extensiones de software completo ofrece dos niveles de extensibilidad que cubren desde guias procedurales simples hasta integraciones complejas con servicios externos. Decidir entre uno y otro depende de la naturaleza de la extension: si el problema se resuelve con instrucciones y contexto, una skill es suficiente; si requiere codigo, acceso a la red o logica de estado, el camino es un plugin.

