# Repositorios de Artefactos

Un repositorio de artefactos es el almacen centralizado donde se almacenan, versionan y distribuyen los productos compilados del proceso de integracion continua: imagenes Docker, paquetes Maven, binarios npm, wheels de Python, o cualquier otro tipo de artefacto generado por el pipeline. Sin un repositorio de artefactos, los pipelines de CI/CD no tienen un punto de referencia estable para los binarios que producen, lo que genera ambiguedades sobre que版本 exacta de un componente esta desplegada en cada entorno. Los repositorios de artefactos actuan como el "almacen" que alimenta toda la cadena de despliegue, garantizando trazabilidad, reproducibilidad y control de acceso sobre los productos del build.

## Contenedores y registros Docker

Las imagenes Docker son el formato de empaquetado dominante para aplicaciones containerizadas, y los registros Docker son los repositorios especificos para almacenar y distribuir estas imagenes. Un registro Docker almacena imagenes organizadas por nombre y tag, y sirve las capas de la imagen via el protocolo de distribution de OCI (Open Container Initiative).

El **Docker Hub** es el registro publico por defecto que Docker utiliza. Contiene imagenes oficiales de sistemas operativos, bases de datos, lenguajes de programacion y herramientas. Aunque es conveniente para desarrollo, confiar exclusivamente en Docker Hub para entornos de produccion presenta riesgos: la disponibilidad depende de un tercero, los limites de descarga pueden afectar los pipelines, y las imagenes publicas pueden contener vulnerabilidades no detectadas.

**Harbor** es un registro de contenedores open source diseñado para instalaciones enterprise. Su valor diferencial radica en la seguridad y el cumplimiento normativo: integra escaneo de vulnerabilidades automatico, politicas de admision que controlan que imagenes pueden desplegarse, registros de auditoria detallados y replicacion entre registros. Cuando un pipeline CI genera una imagen Docker, Harbor puede escanearla automaticamente y bloquear su distribucion si contiene vulnerabilidades criticas.

> Los registros Docker no son simples almacenes de archivos: implementan un protocolo de resolucion de capas que permite descargas incrementales, ya que solo transfieren las capas que el cliente no tiene localmente. Esto es fundamental para la eficiencia en entornos con multiples despliegues.

## Repositorios universales: Nexus y Artifactory

Mientras los registros Docker se especializan en imagenes de contenedores, los repositorios universales manejan multiples formatos de artefactos dentro de una sola plataforma. Esto los convierte en un punto centralizado para todas las necesidades de artefactos de una organizacion.

**Sonatype Nexus Repository** es un repositorio universal que soporta formatos como Maven, npm, PyPI, Docker, Helm, Go y muchos mas. Su modelo organiza los repositorios en tres tipos: proxy (cachea artefactos de repositorios upstream como Maven Central), hosted (almacena artefactos generados internamente) y group (expone una unica URL que combina multiples repositorios). Esta arquitectura permite a los desarrolladores apuntar a una sola URL de configuracion sin preocuparse por la ubicacion fisica de cada artefacto.

**JFrog Artifactory** ofrece funcionalidades similares a Nexus pero con enfasis adicional en la integracion con pipelines de CI/CD. Su caracteristica "Build Info" permite registrar metadatos completos de cada build: que dependencias se usaron, que artefactos se generaron, que vulnerabilidades se detectaron. Esta informacion se almacena permanentemente en Artifactory, creando una linea de trazabilidad completa desde el commit de codigo hasta el artefacto desplegado.

Ambos repositorios implementan politicas de retencion que determinan por cuanto tiempo se conservan los artefactos, cuantos pueden almacenarse y cuales pueden eliminarse automaticamente. Estas politicas son criticas para gestionar el almacenamiento a largo plazo, ya que un repositorio activo puede acumular terabytes de artefactos.

## Integracion con plataformas de codigo

Las plataformas de gestion de codigo como GitHub y GitLab ofrecen registros de artefactos integrados, eliminando la necesidad de infraestructura separada para equipos que buscan simplicidad.

**GitHub Packages** es el servicio de paquetes de GitHub que soporta multiples formatos (npm, Maven, RubyGems, NuGet, Docker). Su principal ventaja es la integracion con los permisos de GitHub: los mismos tokens y control de acceso que gestiona el repositorio se aplican al registro de paquetes. Sin embargo, GitHub Packages tiene limitaciones en cuanto a escala y formato, y puede ser mas costoso que alternativas independientes para volumenes grandes de descarga.

**GitLab Container Registry** es el registro de contenedores integrado en GitLab. Al igual que GitHub Packages, hereda los permisos del proyecto y elimina la necesidad de gestionar credenciales adicionales. Para equipos que ya utilizan GitLab, es la alternativa natural, aunque carece de algunas funcionalidades avanzadas de escaneo y politicas que ofrecen soluciones como Harbor.

La decision entre una solucion integrada y una independiente depende del volumen de artefactos, los requisitos de seguridad y la complejidad de los flujos de distribucion. Para equipos pequenos, la integracion nativa suele ser suficiente. Para organizaciones grandes con multiples equipos y requisitos regulatorios, un repositorio independiente como Nexus o Artifactory ofrece mas control.

> Los repositorios integrados en plataformas de codigo suelen tener limitaciones en cuanto a politicas de retencion y replicacion. Si la organizacion necesita distribuir artefactos a multiples centros de datos o regiones, un repositorio independiente es practicamente obligatorio.

## Versionado semantico de artefactos

El versionado semantico es un estandar para comunicar significado de cambios en una版本 mediante el formato `MAJOR.MINOR.PATCH`. Comprender y aplicar este sistema es fundamental para la gestion de artefactos porque determina como los consumidores de un artefacto interpretan los cambios y cuando actualizan sus dependencias.

**MAJOR** indica cambios incompatibilidades con versiones anteriores. Unaactualizacion de MAJOR puede romper el codigo que consume el artefacto. **MINOR** indica funcionalidad nueva compatible con versiones anteriores. **PATCH** indica correccion de errores sin cambiar la interfaz publica.

En un repositorio de artefactos, las etiquetas de versionado permiten resolver dependencias de forma predecible. Cuando un proyecto depende de `libreria@^2.1.0`, el gestor de paquetes entiende que puede instalar cualquier版本 2.x.x igual o superior a 2.1.0, pero no 3.0.0. Este contrato implicito es lo que permite que los pipelines de CI generen artefactos con versiones que los consumidores pueden adoptar gradualmente.

Los repositorios de artefactos tipicamente soportan multiples estrategias de versionado: versionado semantico estricto, versionado con hashes de commit (para artefactos sin版本 formal), y versionado por rama o tag de Git (util para artefactos en desarrollo continuo). La consistencia en la estrategia de versionado es critica para evitar ambiguedades en el despliegue.

> Un error comun es sobre-utilizar versiones pre-release (ej: `1.0.0-beta.1`) en entornos de produccion. Los repositorios de artefactos deben configurar politicas claras sobre que etiquetas de versionado son aptas para cada entorno.

## El repositorio como pilar del pipeline

El repositorio de artefactos no es un componente opcional en una pipeline CI/CD: es el contrato entre la fase de construccion y la fase de despliegue. Cuando la CI genera un artefacto, lo registra en el repositorio con metadatos que incluyen el commit de origen, la identidad del build y los resultados de las pruebas. Cuando la CD despliega un artefacto, lo obtiene del repositorio verificando su integridad y version.

Esta separacion de responsabilidades permite que la CI y la CD operen de forma independiente pero coordinada. La CI puede generar multiples versiones de un artefacto sin afectar el despliegue, y la CD puede desplegar cualquier版本 previamente validada sin necesidad de reconstruir. Esta indireccion es la que permite funcionalidades criticas como rollback instantaneo, despliegue por tiempo (time-based deployment) y comparacion directa entre versiones.

Sin un repositorio de artefactos, los pipelines dependen de reconstrucciones repetidas desde el codigo fuente, lo que introduce riesgo de no-reproducibilidad: las dependencias externas pueden cambiar, los entornos de build pueden diferir, y los artefactos resultantes pueden no ser identicos. El repositorio garantiza que el artefacto desplegado es exactamente el mismo que fue validado en la fase de integracion.

