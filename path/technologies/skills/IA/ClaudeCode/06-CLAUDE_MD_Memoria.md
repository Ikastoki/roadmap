# Claude Code CLI

Cada proyecto de software desarrolla con el tiempo un conjunto implicito de convenciones, preferencias arquitectonicas y reglas operativas que los miembros del equipo internalizan pero que rara vez estan formalizadas en un lugar accesible para herramientas automatizadas. El archivo `CLAUDE.md` cierra esa brecha: actua como un manifiesto de intenciones que Claude Code lee al iniciar cada sesion, estableciendo el contexto permanente del proyecto sin necesidad de repetir instrucciones en cada interaccion. A este mecanismo se suman la memoria automatica, que preserva decisiones y configuraciones relevantes entre sesiones, y las instrucciones personalizadas que permiten extender el comportamiento del agente mediante argumentos de linea de comandos.

## CLAUDE.md

`CLAUDE.md` es un archivo Markdown ubicado en la raiz del proyecto que contiene instrucciones personalizadas para guiar el comportamiento del agente. Claude Code lo detecta automaticamente al iniciar una sesion en el directorio del proyecto y carga su contenido como parte del contexto fundamental del agente. No es un archivo de configuracion tecnica al uso, sino un documento en lenguaje natural donde se definen convenciones de codigo, estructura del proyecto, preferencias de frameworks, configuracion de herramientas de testing, politicas de seguridad y cualquier otra directriz que deba aplicarse consistentemente en todas las sesiones y por todos los miembros del equipo.

El comando `/init` crea o actualiza `CLAUDE.md` de forma interactiva: Claude Code escanea la estructura del proyecto, identifica el lenguaje de programacion principal, el gestor de paquetes, el framework de testing y otras caracteristicas relevantes, y genera un conjunto inicial de reglas adaptadas al contexto especifico del proyecto. El usuario puede aceptar, modificar o extender cada seccion durante el proceso interactivo, resultando en un archivo listo para usar que refleja tanto las practicas detectadas como las preferencias explicitas del equipo.

```bash
# Iniciar la creacion interactiva de CLAUDE.md
claude --init

# El agente analiza el proyecto y propone reglas
# El usuario confirma o ajusta cada seccion
```

La ubicacion predeterminada es la raiz del proyecto (`./CLAUDE.md`). Claude Code tambien reconoce `AGENTS.md` como alternativa compatible con OpenCode, permitiendo que proyectos que ya utilizan esta convencion no necesiten renombrar sus archivos de instrucciones. Ademas, existe un archivo global en `~/.claude/CLAUDE.md` que contiene instrucciones aplicables a todos los proyectos. La precedencia es clara: las reglas del archivo local (proyecto) tienen prioridad sobre las del archivo global, y si ambos existen, se fusionan resolviendo los conflictos a favor del proyecto.

A continuacion se muestra un ejemplo completo de `CLAUDE.md` que ilustra las secciones tipicas y el nivel de detalle recomendado:

```markdown
# Reglas del proyecto

Eres un asistente de codigo para la API de pagos.

## Convenciones de codigo

- TypeScript con strict mode, imports con path alias `@/`
- Funciones flecha, nombres en camelCase, interfaces con prefijo `I`
- Tests en Vitest con descripciones en español
- Commits convencionales siguiendo Conventional Commits

## Estructura del proyecto

src/
  api/       -> controladores y rutas Express
  domain/    -> entidades y logica de negocio
  infra/     -> repositorios, conexiones a DB, servicios externos
  shared/    -> tipos, utilidades, constantes
tests/
  unit/      -> tests por modulo
  integration/ -> tests con base de datos real

## Testing

- Todos los modulos nuevos deben tener cobertura > 80%
- Usar `npx vitest run` para ejecutar tests
- Los tests de integracion requieren PostgreSQL en local
- Mockear siempre las llamadas HTTP externas con msw

## Seguridad

- No almacenar secretos en el codigo, usar variables de entorno
- Validar toda entrada con zod antes de procesarla
- Sanitizar salidas en respuestas HTTP
- logging estructurado con pino, nunca console.log

## Notas tecnicas

- La base de datos se migra con drizzle-kit
- El despliegue usa Docker Compose con tres servicios: api, worker, db
- Las claves de API se gestionan via el servicio interno de secrets
```

## Auto Memory

La memoria automatica es una caracteristica que permite a Claude Code retener informacion relevante entre sesiones sin intervencion manual del usuario. Cuando el agente detecta datos que probablemente seran utiles en el futuro --una decision arquitectonica importante, la configuracion de una herramienta, una preferencia de estilo expresada durante la conversacion-- los persiste en `CLAUDE.md` de forma estructurada. Este mecanismo evita que el usuario tenga que repetir informacion contextual en cada nueva sesion y construye progresivamente un perfil del proyecto que se vuelve mas preciso con el tiempo.

El comando `/memory` proporciona control explicito sobre esta memoria. Permite listar los recuerdos almacenados, anadir entradas manualmente, editar o eliminar informacion persistida, y revisar el historial de cambios en la memoria del proyecto. Es el punto de entrada para la gestion consciente de la memoria, complementando la recoleccion automatica que realiza el agente.

```bash
# Listar entradas de memoria del proyecto
/memory list

# Anadir una entrada manual
/memory add "La API de pagos usa stripe como proveedor principal"

# Eliminar una entrada especifica
/memory delete "stripe-config"
```

La diferencia fundamental entre la memoria automatica y las reglas explicitas en `CLAUDE.md` radica en su origen y proposito. Las reglas explicitas son directrices deliberadas, escritas y mantenidas por el equipo de desarrollo, que definen politicas estables del proyecto. La memoria automatica, en cambio, es derivada: Claude Code infiere que cierta informacion es relevante basandose en las interacciones y la persigue sin necesidad de que el usuario la formalice. Ambas convergen en el mismo archivo `CLAUDE.md`, pero mientras las reglas explicitas raramente cambian, la memoria automatica se actualiza continuamente a medida que el agente descubre nuevo contexto relevante.

## Custom Instructions

Las instrucciones personalizadas permiten extender o modificar el comportamiento de Claude Code sin modificar `CLAUDE.md`. Se proporcionan en el momento de iniciar la sesion mediante dos flags de linea de comandos. `--append-system-prompt` acepta texto inline que se anade directamente al system prompt del modelo. `--append-system-prompt-file` acepta la ruta a un archivo cuyo contenido se incorpora al prompt del sistema. Ambos mecanismos son aditivos: el contenido se fusiona con el system prompt por defecto y con las reglas de `CLAUDE.md`, sin reemplazar ninguno de ellos.

```bash
# Instruccion inline
claude --append-system-prompt "Responde siempre en espanol"

# Instrucciones desde archivo
claude --append-system-prompt-file ./instrucciones_equipo.md

# Combinacion de ambos
claude --append-system-prompt "Prioriza la legibilidad sobre la eficiencia" \
       --append-system-prompt-file ./contexto_legal.md
```

La fusion sigue un orden de precedencia preciso. El system prompt base de Claude Code constituye la capa fundamental con las capacidades nucleares del agente. Sobre el se aplican las reglas de `CLAUDE.md` del proyecto, que establecen el contexto especifico. A continuacion se incorpora `AGENTS.md` si existe, como capa de compatibilidad con proyectos que utilizan ese formato. Las instrucciones via `--append-system-prompt` o `--append-system-prompt-file` constituyen la capa mas externa, con la maxima prioridad. Este orden garantiza que las directrices explicitas del usuario en el momento de la invocacion puedan anular cualquier configuracion preexistente, pero sin perder las capacidades base del agente ni el contexto del proyecto.

> Las instrucciones personalizadas no son persistentes entre sesiones. Cada invocacion de `claude` con estos flags genera una configuracion temporal que solo afecta a la sesion en curso. Para instrucciones permanentes, debe utilizarse `CLAUDE.md`.

La combinacion de estos tres mecanismos --`CLAUDE.md` como memoria estructural del proyecto, la memoria automatica como capa adaptativa que se refina con el uso, y las instrucciones personalizadas como configuracion efimera para necesidades puntuales-- constituye un sistema de contexto multifacetico. El proyecto define su identidad y reglas en `CLAUDE.md`. El agente aprende y retiene informacion relevante a traves de la memoria automatica. El usuario inyecta directrices temporales cuando una sesion concreta requiere un comportamiento especifico. Juntos, eliminan la friccion de tener que reestablecer el contexto en cada interaccion, acercando la experiencia de uso a la de un colaborador que conoce el proyecto tanto como sus miembros humanos.

