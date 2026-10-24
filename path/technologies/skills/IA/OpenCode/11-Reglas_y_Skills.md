# OpenCode CLI

Los proyectos de software no son todos iguales y cada equipo establece sus propias convenciones de codigo, estructura de directorios, procesos de revision y requisitos de calidad. OpenCode resuelve esta variabilidad mediante un sistema de instrucciones personalizadas que pueden definirse a nivel de proyecto, de usuario o incluso de subdirectorio, permitiendo que el agente se comporte de forma coherente con las expectativas del equipo sin necesidad de recordarle explicitamente las reglas en cada interaccion. Este sistema se articula en torno a tres mecanismos: el archivo `AGENTS.md` como fuente principal de reglas, la clave `instructions` en la configuracion para cargar fuentes adicionales y las Skills como modulos reutilizables de conocimiento especializado.

## AGENTS.md

El archivo `AGENTS.md` es el mecanismo fundamental para comunicar al agente las normas, convenciones y restricciones de un proyecto. Cuando OpenCode inicia una sesion, busca este archivo en la raiz del proyecto y lo incorpora como parte del contexto que recibe el modelo al comienzo de cada interaccion. El contenido tipico de `AGENTS.md` incluye convenciones de codigo (estilo de named parameters, uso de archivos de indizacion, estructura de importaciones), requisitos de verificacion (lanzar linter antes de cada commit, mantener cobertura de tests por encima de un umbral, formato de mensajes de commit) y reglas especificas del proyecto (como evitar ciertas dependencias, usar una libreria concreta para logging o seguir un patron de diseno determinado).

OpenCode ofrece el comando `/init` para crear o actualizar `AGENTS.md` de forma interactiva. Al ejecutarlo, la herramienta escanea la estructura del proyecto -- examina el gestor de paquetes, el lenguaje principal, el framework, las herramientas de testing y las dependencias -- y genera un conjunto de reglas adaptadas al stack detectado. El usuario puede revisar, modificar y aprobar cada seccion antes de que se escriba el archivo final. Este proceso garantiza que el `AGENTS.md` resultante refleje tanto las caracteristicas tecnicas del proyecto como las preferencias del equipo.

```md
# AGENTS.md

## Coding conventions
- Use snake_case for variables and functions
- Use PascalCase for classes and interfaces
- All public functions MUST have type hints
- Use `pathlib` instead of `os.path` for file operations
- Maximum line length: 100 characters
- Single quotes for strings, double quotes for docstrings

## Testing requirements
- Every module MUST have a corresponding test file in `tests/`
- Run `pytest --cov --cov-fail-under=80` before each commit
- Use `pytest.fixtures` for test setup, avoid `setUp` methods
- Mock external APIs in integration tests with `responses` library

## Project-specific rules
- Commit messages MUST follow Conventional Commits: `type(scope): message`
- Branch naming: `feature/`, `fix/`, `chore/` followed by issue number
- Never commit directly to `main` or `develop` branches
- Use `loguru` for logging, never `print` or `logging`
- Configuration files live in `config/` directory as YAML
- Database migrations must be reviewed by a second developer
```

El archivo `AGENTS.md` puede ubicarse en dos niveles. El nivel de proyecto, en la raiz del repositorio, se versiona y comparte con todo el equipo, garantizando que todos los desarrolladores que trabajen sobre el codigo reciban las mismas directrices. El nivel global, en `~/.config/opencode/AGENTS.md`, contiene reglas personales que el usuario desea aplicar en todos los proyectos sin excepcion, como preferencias de formato, herramientas habituales o restricciones de seguridad. OpenCode fusiona ambos: las reglas del proyecto se cargan primero y las reglas globales se anexan a continuacion, con las reglas locales prevaleciendo en caso de conflicto.

OpenCode es compatible con `CLAUDE.md`, el archivo de instrucciones utilizado por Claude Code de Anthropic. Si el proyecto contiene un `CLAUDE.md` pero no un `AGENTS.md`, OpenCode lo lee como fuente de instrucciones. Si ambos existen, se fusionan siguiendo la precedencia: `AGENTS.md` del proyecto (maxima prioridad), `CLAUDE.md` del proyecto, `~/.config/opencode/AGENTS.md` global y finalmente `~/.claude/CLAUDE.md` global. Esta compatibilidad permite que los equipos que migran desde Claude Code mantengan su configuracion existente mientras adoptan progresivamente las capacidades especificas de OpenCode.

## Instrucciones adicionales

La clave `instructions` en `opencode.json` permite anadir fuentes de instrucciones complementarias al `AGENTS.md`. Cada entrada puede ser una ruta local a un archivo Markdown o una URL remota. OpenCode descarga y procesa estas fuentes durante la inicializacion y las incorpora al contexto del modelo como si fueran parte del `AGENTS.md`. Este mecanismo es util para incluir documentacion generada automaticamente, guias de estilo corporativas alojadas en un repositorio central o politicas de seguridad mantenidas por un equipo de infraestructura.

El campo `instructions` soporta patrones glob, lo que permite incluir archivos de multiples subdirectorios con una sola entrada. Por ejemplo, `packages/*/AGENTS.md` carga los archivos de instrucciones de cada paquete dentro de un monorepositorio, permitiendo que cada subsistema defina sus propias reglas sin interferir con las de los demas. OpenCode resuelve los globs en el momento de la inicializacion y procesa los archivos encontrados en orden lexicografico, garantizando un orden predecible en la fusion de instrucciones.

```json
{
  "instructions": [
    "docs/STYLE_GUIDE.md",
    "https://empresa.com/politicas/seguridad.md",
    "packages/*/AGENTS.md"
  ]
}
```

Las instrucciones provenientes de `instructions` se fusionan con el `AGENTS.md` siguiendo una concatenacion simple: primero se carga el `AGENTS.md` del proyecto, luego los archivos listados en `instructions` en el orden en que aparecen, y finalmente el `AGENTS.md` global. No existe deduplicacion automatica: si dos fuentes definen reglas contradictorias, la ultima en aparecer prevalece. Esto otorga al desarrollador control total sobre el orden de precedencia, ya que puede reordenar las entradas de `instructions` segun sus necesidades.

## Skills (SKILL.md)

Las Skills son unidades reutilizables de instrucciones que el agente carga bajo demanda, a diferencia del `AGENTS.md` que se inyecta siempre en cada sesion. Una Skill reside en `~/.config/opencode/skills/<nombre>/SKILL.md` (global) o en `.opencode/skills/<nombre>/SKILL.md` (por proyecto) y contiene un bloque YAML de frontmatter con metadatos seguido del cuerpo de instrucciones en Markdown. Cuando la herramienta `skill` determina que el contexto actual requiere conocimientos especializados -- como el procedimiento de despliegue en Kubernetes, la configuracion de un linter personalizado o el uso de una API interna -- carga la Skill correspondiente y la inyecta en el contexto del modelo.

La diferencia fundamental entre `AGENTS.md` y una Skill reside en el momento de activacion y el ambito. `AGENTS.md` es omnipresente: sus reglas estan siempre disponibles y definen el comportamiento basal del agente en el proyecto. Una Skill, en cambio, es un modulo optativo que solo se activa cuando el agente lo necesita, mediante la herramienta `skill` que evalua la descripcion de la Skill contra la tarea actual. Esto permite construir bibliotecas extensas de conocimiento especializado sin saturar el contexto del modelo con informacion irrelevante para la mayoria de las interacciones.

```yaml
---
name: deploy-kubernetes
description: Procedimiento de despliegue en Kubernetes para el equipo de plataforma
license: MIT
compatibility:
  opencode: ">=0.15.0"
  kubernetes: ">=1.25"
---

# Deploy a Kubernetes

## Prerequisitos
- `kubectl` configurado con el contexto del cluster de destino
- Acceso al registry de imagenes del proyecto
- Helm 3 instalado

## Pasos
1. Construir la imagen: `docker build -t registry.internal/app:{{version}} .`
2. Publicar la imagen: `docker push registry.internal/app:{{version}}`
3. Actualizar la helm chart: `helm upgrade app ./helm/app --set image.tag={{version}}`
4. Verificar el despliegue: `kubectl rollout status deployment/app`

## Rollback
Si el despliegue falla, ejecutar `helm rollback app 0` para volver a la revision anterior.

## Notas
- Las variables de entorno del cluster se gestionan via External Secrets Operator
- No modificar manualmente los Secrets de Kubernetes
```

Las Skills se controlan mediante patrones de permiso bajo la clave `skill` en la configuracion. El sistema permite autorizar o denegar Skills completas, grupos por patron de nombre o Skills individuales. Un caso de uso tipico es permitir todas las Skills de codigo abierto pero denegar las internas que exponen infraestructura sensible: `{"*": "allow", "internal-*": "deny"}`. OpenCode evalua las reglas en orden de especificidad: los patrones mas concretos tienen prioridad sobre los mas generales. Si no se define ninguna regla, el comportamiento por defecto es `"ask"`, solicitando confirmacion al usuario antes de cargar la Skill.

Las Skills pueden compartirse entre equipos alojandolas en repositorios Git. Un equipo puede mantener un repositorio central de Skills con procedimientos estandarizados para testing, despliegue, auditoria de seguridad y generacion de documentacion. OpenCode permite referenciar Skills mediante URL de Git, clonando el repositorio en el momento de la primera solicitud y cacheandolo localmente. Este modelo convierte las Skills en un mecanismo de gobernanza: los equipos de plataforma pueden definir y distribuir procedimientos obligatorios que los desarrolladores consumen sin esfuerzo directamente desde el agente.

> A diferencia de `AGENTS.md`, cuyas reglas se aplican implicitamente en cada sesion, una Skill solo se activa cuando la herramienta `skill` decide que es relevante. Esto implica que las reglas criticas de seguridad o compliance deben residir en `AGENTS.md`, no en una Skill, para garantizar que siempre esten presentes.

