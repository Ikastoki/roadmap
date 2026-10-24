# Claude Code CLI

## Dynamic Workflows

Un workflow dinamico es el mecanismo mediante el cual Claude Code descompone un problema de gran envergadura en unidades logicamente independientes y las distribuye entre multiples subagentes que operan en paralelo. A diferencia de un proceso secuencial donde el agente principal aborda cada paso de forma lineal, los workflows dinamicos introducen paralelismo real: el agente orquestador analiza el problema, identifica las fronteras naturales de separacion, genera un plan con unidades de trabajo autocontenidas y lanza tantos subagentes como unidades existan, cada uno en un contexto aislado. Este enfoque reduce el tiempo de ejecucion de tareas masivas de horas a minutos, siempre que las unidades sean verdaderamente independientes y no compartan estado mutable.

El comando `/batch` materializa este concepto para cambios masivos sobre el codigo fuente. Al invocarlo, Claude Code realiza una fase de reconocimiento del arbol del proyecto, identifica entre 5 y 30 unidades de trabajo (por ejemplo, archivos a refactorizar, funciones a documentar o modulos a migrar) y presenta al usuario un plan estructurado con el alcance de cada unidad. Una vez aprobado el plan, el agente lanza un subagente background por cada unidad, ejecutandose cada uno en un worktree de git aislado. El aislamiento mediante worktree es crucial: garantiza que los cambios en una unidad no interfieran con los de otra, eliminando conflictos de edicion concurrente sobre el mismo repositorio. Cada subagente opera en su propia rama y, al finalizar, el workflow integra los resultados mediante merges controlados.

```bash
/batch Refactorizar todas las funciones de validacion para usar el nuevo schema de Zod
```

```bash
/batch Anadir tests unitarios a todos los controladores de la carpeta src/controllers/
```

`/deep-research` traslada el mismo principio de paralelismo al dominio de la investigacion. Ante una pregunta compleja, el agente formula multiples consultas de busqueda web independientes, las lanza simultaneamente, cruza los resultados de distintas fuentes, identifica discrepancias y puntos de consenso, y sintetiza un informe estructurado con referencias cruzadas. A diferencia de una busqueda unica, la descomposicion en consultas paralelas permite cubrir multiples angulos del problema simultaneamente -- documentacion oficial, analisis de seguridad, comparativas de rendimiento, casos de uso reales -- y producir una respuesta que ningun resultado individual contenia por si solo.

```bash
/deep-research Comparativa de frameworks de testing en Rust: criterios de rendimiento, ecosistema y madurez
```

> El plan generado por `/batch` puede rechazarse, modificarse o aprobarse parcialmente. Si se rechaza, el workflow se aborta sin efectos secundarios; si se modifica, el agente ajusta las unidades segun las indicaciones del usuario antes de lanzar los subagentes.

## Gestion de workflows

El comando `/workflows` proporciona un panel de control unificado para supervisar y manipular los workflows en ejecucion. Desde el se puede pausar un workflow en curso, reanudarlo cuando las condiciones sean favorables, cancelar unidades especificas sin abortar el workflow completo, o inspeccionar el estado de cada subagente individual. La visibilidad granular que ofrece este comando es esencial cuando se gestionan workflows con decenas de subagentes ejecutandose simultaneamente, permitiendo identificar rapidamente que unidades han fallado, cuales estan pendientes y cuales se estan ejecutando en este momento.

```bash
/workflows            # Listar workflows activos con su estado y progreso
/workflows pause      # Pausar todos los workflows en ejecucion
/workflows resume     # Reanudar workflows pausados
```

El tamano de un workflow determina el numero maximo de subagentes que pueden ejecutarse concurrentemente. Los modos disponibles son `small` (pocos subagentes, adecuado para tareas ligeras con pocas unidades), `medium` (menos de 15 subagentes, el equilibrio recomendado para la mayoria de refactorizaciones), `large` (escalado agresivo para proyectos extensos) y `unrestricted` (sin limite superior, bajo responsabilidad del usuario). La eleccion del tamano impacta directamente en el consumo de recursos del sistema y en la presion sobre la tasa de uso de la API de Claude. Para la mayoria de las tareas, `medium` ofrece el mejor balance entre paralelismo y estabilidad.

El control de profundidad limita el anidamiento de subagentes, evitando que un subagente lance a su vez otros subagentes en una cascada incontrolada. El valor por defecto es 3 niveles de profundidad, configurable mediante la variable de entorno `CLAUDE_CODE_MAX_DEPTH`. Un valor de 1 restringe la ejecucion a un solo nivel de subagentes sin anidamiento, mientras que valores superiores a 3 permiten arboles de delegacion profundos utiles en tareas que requieren descomposicion recursiva del problema.

```bash
export CLAUDE_CODE_MAX_DEPTH=5
claude
```

```bash
# Workflow large con profundidad personalizada
CLAUDE_CODE_MAX_DEPTH=4 claude --model claude-opus-4-20260514
```

Los subagentes dentro de un workflow pueden ejecutarse en dos modalidades: foreground y background. Los subagentes foreground reportan su progreso directamente en la conversacion principal, mostrando en tiempo real las decisiones que toman, los archivos que modifican y los errores que encuentran. Esta modalidad es util para workflows pequenos donde se desea seguimiento detallado. Los subagentes background, en cambio, trabajan en segundo plano sin saturar la conversacion; solo notifican eventos relevantes (finalizacion, error critico, solicitud de confirmacion) y pueden monitorizarse a traves del panel de Agent View.

## Agent View

`claude agents` abre un panel de monitorizacion visual que lista todas las sesiones background activas en el sistema. Cada sesion aparece con su identificador unico, el workflow al que pertenece, el estado actual (active, completed, failed), el tiempo transcurrido y el comando que esta ejecutando. El panel se actualiza en tiempo real y permite inspeccionar el log de cada sesion, cancelar sesiones individuales o limpiar sesiones completadas. Es la herramienta principal para mantener el control sobre multiples subagentes cuando se gestionan workflows de gran escala.

```bash
claude agents                # Panel interactivo de monitorizacion
claude agents --json         # Salida JSON para scripting y automatizacion
claude agents --cwd /ruta/del/proyecto  # Filtrar sesiones por directorio
claude agents --all          # Incluir sesiones completadas en la lista
```

La opcion `--json` estructura la salida como un array JSON con todos los campos de cada sesion, permitiendo integrar la monitorizacion en pipelines de CI/CD, scripts de notificacion o dashboards personalizados. `--cwd <path>` filtra las sesiones que se ejecutan dentro de un directorio concreto, util cuando se trabaja con multiples proyectos simultaneamente. `--all` extiende la vista para incluir sesiones completadas y fallidas, no solo las activas, facilitando la revision post-mortem de workflows anteriores.

Por detras de estas capacidades opera el supervisor de background sessions, un proceso守护 que gestiona el ciclo de vida completo de todas las sesiones background: las crea, monitoriza su consumo de recursos, reinicia sesiones fallidas cuando la politica lo permite y recolecta los resultados al finalizar. Este supervisor es el responsable de mantener el aislamiento entre sesiones, de liberar los worktree de git cuando una sesion termina y de gestionar la cola de ejecucion cuando se supera el limite de concurrencia definido por el tamano del workflow. Sin el supervisor, la ejecucion paralela de decenas de subagentes seria simplemente inviable, pues no existiria una entidad central capaz de coordinar recursos, resolver contención y garantizar que cada subagente complete su cometido dentro de los margenes de aislamiento y seguridad que exige un workflow de produccion.

```bash
# Consultar el estado del supervisor
claude agents --json | jq '.[] | {id, status, elapsed_seconds, workflow}'
```

