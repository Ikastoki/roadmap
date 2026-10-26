# Herramientas de Integracion Continua

La integracion continua no es un concepto abstracto: requiere de herramientas concretas que ejecuten los pasos automatizados de compilacion, prueba y validacion cada vez que un desarrollador introduce un cambio en el repositorio. La eleccion de la herramienta CI adecuada depende de multiples factores: donde se aloja el codigo fuente, el tamano del equipo, los requisitos de infraestructura, el presupuesto disponible y el nivel de control que se necesita sobre el entorno de ejecucion. Comprender las fortalezas y limitaciones de cada alternativa permite tomar decisiones informadas que impactan directamente en la velocidad y fiabilidad del ciclo de desarrollo.

## GitHub Actions

GitHub Actions es el sistema de CI/CD nativo de GitHub, lo que le otorga una ventaja competitiva significativa para equipos que ya utilizan esa plataforma. La integracion es total: los workflows se definen en archivos YAML dentro del repositorio (`.github/workflows/`) y se ejecutan automaticamente en respuesta a eventos del repositorio como `push`, `pull_request` o `release`. No existe necesidad de configurar webhooks externos ni gestionar servidores de integracion separados.

El ecosistema de acciones disponibles en el [GitHub Marketplace](https://github.com/marketplace?type=actions) permite reutilizar componentes preconstruidos para tareas comunes como despliegues a cloud, ejecucion de tests o notificaciones. Cada workflow se compone de jobs que corren en containers o VMs, y estos pueden ejecutarse en paralelo o de forma secuencial segun las dependencias definidas.

Una caracteristica relevante es el sistema de secrets, que permite almacenar credenciales de forma segura y referenciarlas dentro de los workflows sin exponerlas en el historial de versiones. Para proyectos open source, GitHub ofrece minutos de ejecucion gratuitos, lo cual reduce significativamente la barrera de entrada.

```yaml
# Ejemplo simplificado de un workflow de GitHub Actions
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

> GitHub Actions permite definir matrices de ejecucion para probar contra multiples versiones de un lenguaje o sistema operativo en paralelo, algo particularmente util para librerias que deben mantener compatibilidad con diferentes entornos.

## GitLab CI/CD

GitLab CI/CD es el sistema de integracion y despliegue continuo integrado en la plataforma GitLab. Al igual que GitHub Actions, se configura mediante un archivo YAML (`.gitlab-ci.yml`) en la raiz del repositorio, pero ofrece una filosofia ligeramente diferente: GitLab concibe CI/CD como una parte inseparable del ciclo de vida del software, no como un complemento.

Los pipelines de GitLab se organizan en etapas (`stages`) y los trabajos (`jobs`) dentro de cada etapa se ejecutan en paralelo. Las etapas definen un orden de ejecucion: si todos los jobs de una etapa terminan correctamente, se avanza a la siguiente. Esto crea un modelo mental claro del flujo de datos a traves del pipeline.

GitLab incluye un feature propio llamado Auto DevOps que detecta automaticamente el tipo de proyecto y configura un pipeline predeterminado con mejores practicas, reduciendo la configuracion manual para proyectos estandar. Tambien integra de forma nativa un container registry, un package registry y capacidades de revision de codigo (merge requests), creando un ecosistema cerrado pero coherente.

Para instalaciones self-hosted, GitLab Runner es un agente que ejecuta los jobs del pipeline. Puede configurarse con ejecutadores Docker, Shell, o Kubernetes, ofreciendo flexibilidad en cuanto al entorno de ejecucion.

> GitLab distingue entre pipelines para la rama principal, merge requests y ramas temporales, permitiendo configurar diferentes niveles de intensidad de pruebas segun el contexto.

## Jenkins

Jenkins representa la alternativa madura y altamente configurable para equipos que necesitan control total sobre su infraestructura de CI/CD. A diferencia de las soluciones SaaS, Jenkins es un servidor self-hosted que se ejecuta en infraestructura propia, lo que implica tanto mayor control como mayor responsabilidad en su mantenimiento.

Su arquitectura se basa en un maestro con multiples agentes (nodos) que ejecutan los trabajos distribuidos. El maestro coordina la ejecucion, almacena los resultados y sirve la interfaz web de gestion. Los agentes pueden ser maquinas fisicas, virtuales o containers, conectados via SSH o JNLP.

El ecosistema de plugins de Jenkins es vasto (mas de 1800 plugins disponibles) y cubre praticamente cualquier herramienta, plataforma o flujo de trabajo imaginable. Esta extensibilidad es tanto su mayor fortaleza como su talon de Aquiles: un Jenkins con demasiados plugins puede volverse inestable y dificil de mantener. La deuda tecnica acumulada en instalaciones antiguas es un problema comun en la industria.

La configuracion de Jenkins se realiza principalmente a traves de la interfaz web o mediante archivos `Jenkinsfile` escritos en Groovy. El pipeline declarativo, introducido mas recientemente, ofrece una sintaxis mas estructurada y legible que el pipeline tradicional basado en scripts Groovy.

> Jenkins requiere dedicacion continua de recursos para actualizaciones, parches de seguridad y mantenimiento de plugins. Es una solucion poderosa, pero no gratuita en terminos de esfuerzo operativo.

## Servicios SaaS: CircleCI y TravisCI

Las plataformas SaaS de CI/CD eliminan la carga de gestionar infraestructura propia. CircleCI y TravisCI son dos representantes consolidados de este modelo, cada uno con su enfoque particular.

**CircleCI** se destaca por su modelo de ejecucion basado en containers, lo que permite pipelines de arranque rapido (el primer job puede comenzar en segundos). Ofrece configuracion via el archivo `.circleci/config.yml` y permite paralelizar la ejecucion de tests dividiendolos en multiples containers. Su modelo de costes se basa en minutos de ejecucion consumidos, con un tier gratuito limitado.

**TravisCI** fue uno de los primeros servicios de CI en integrarse directamente con GitHub y mantiene una interfaz sencilla para proyectos open source. Su archivo de configuracion (`.travis.yml`) es conciso y facil de entender para casos simples. Sin embargo, TravisCI ha perdido cuota de mercado en los ultimos anos debido a cambios en su modelo de negocio y problemas de confiabilidad en su plataforma.

Ambas plataformas ofrecen integraciones con herramientas de notificacion, despliegue a multiples proveedores cloud y caching de dependencias para acelerar las ejecuciones. La decision entre SaaS y self-hosted generalmente se reduce a una balanza entre comodidad y control: SaaS elimina la carga operativa pero limita la personalizacion del entorno de ejecucion.

> Los servicios SaaS pueden introducir dependencia del proveedor. Si la plataforma sufre un incidente o cambia su modelo de precios, el pipeline completo de la organizacion se ve afectado.

## Criterios de seleccion

La eleccion entre herramientas de CI no es universal y depende del contexto especifico de cada equipo. Los factores clave a considerar incluyen:

**Ecosistema existente**: Si el equipo utiliza GitHub, GitHub Actions ofrece la menor friccion de integracion. Si utiliza GitLab, su CI/CD nativo es la opcion mas coherente. Introducir una herramienta externa增加了 complejidad sin beneficio claro.

**Tamano del equipo y madurez**: Equipos pequenos y startups se benefician de soluciones SaaS con minima configuracion. Equipos grandes con requisitos de compliance o seguridad especificos pueden necesitar soluciones self-hosted como Jenkins o GitLab Runner.

**Presupuesto**: Las soluciones SaaS tienen costes recurrentes basados en uso. Jenkins es gratuito pero requiere personal para su gestion. GitHub Actions y GitLab CI ofrecen tiers gratuitos generosos para proyectos open source.

**Control y personalizacion**: Cuando se necesitan flujos de trabajo complejos, integraciones con sistemas legacy o ejecucion en entornos restringidos, Jenkins sigue siendo difficult de superar en flexibilidad.

**Requisitos de seguridad**: Algunas organizaciones prohiben que el codigo fuente salga de su red. En estos casos, solo las soluciones self-hosted (Jenkins, GitLab Runner) son viables.

| Herramienta | Modelo | Integracion nativa | Complejidad operativa |
|---|---|---|---|
| GitHub Actions | SaaS / self-hosted | GitHub | Baja |
| GitLab CI/CD | SaaS / self-hosted | GitLab | Baja-Media |
| Jenkins | Self-hosted | Cualquier SCM | Alta |
| CircleCI | SaaS | GitHub, Bitbucket | Nula |
| TravisCI | SaaS | GitHub | Nula |

