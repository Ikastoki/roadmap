# Secretos en CI/CD

Los pipelines de integracion continua y despliegue continuo (CI/CD) representan uno de los objetivos mas atractivos para atacantes porque concentran acceso a multiples entornos, sistemas de despliegue, repositorios y credenciales de servicios externos en un solo punto de ejecucion. Un pipeline bien configurado necesita acceso a secretos para compilar, testear y desplegar el codigo, pero cada secreto inyectado en el pipeline amplia la superficie de ataque. Equilibrar funcionalidad y seguridad en este contexto es uno de los desafios centrales de la ingenieria de seguridad moderna.

## Por que los pipelines son objetivos prioritarios

Un pipeline de CI/CD tipicamente necesita credenciales para acceder al repositorio de codigo, al registro de contenedores, al cluster de Kubernetes, a la base de datos de staging, al servicio de monitoreo, al proveedor de nube, al repositorio de artefactos y a multiples servicios de terceros. Esto significa que un solo pipeline puede contener simultaneamente docenas de secretos que, de forma aislada, protegen sistemas criticos pero que juntos representan un "botes de tesoros" para cualquier atacante que logre ejecutar codigo dentro del entorno de ejecucion del pipeline.

Los pipelines ejecutan codigo de terceros (acciones de GitHub Actions, plugins de Jenkins, templates de GitLab) que pueden contener vulnerabilidades o comportamientos maliciosos. Un supply chain attack que comprometa una dependencia del pipeline puede robar todos los secretos inyectados en la sesion de ejecucion. Esto ha ocurrido en la practica con compromisos de paquetes npm, GitHub Actions populares y plugins de Jenkins.

Ademas, los logs de los pipelines son un vector de filtracion frecuente. Si un secreto se imprime accidentalmente en un log (por un error en un script, un echo de depuracion o una variable de entorno que se registra junto con el output), queda almacenado en el historial de ejecuciones del pipeline, accesible a cualquiera con permisos de lectura.

## Variables de entorno en pipelines

El mecanismo mas basico y comun para inyectar secretos en pipelines es el uso de variables de entorno. Los sistemas de CI/CD como GitHub Actions, GitLab CI, Jenkins y CircleCI proporcionan mecanismos nativos para definir variables secretas que se inyectan como variables de entorno durante la ejecucion del job.

### Funcionamiento

El concepto es simple: el operador define la variable en la interfaz de la plataforma CI/CD, la marca como "secreta" para que no se muestre en la interfaz ni en los logs, y el pipeline la referencia como `$SECRET_NAME` en los scripts. Internamente, la plataforma almacena el valor cifrado y lo descifra solo durante la ejecucion, pasandolo como variable de entorno al proceso que ejecuta el job.

Cada plataforma tiene matices distintos. GitHub Actions permite definir secrets a nivel de repositorio, de organizacion o de environment (con protecciones adicionales como revisiones de aprobacion). GitLab CI soporta variables protegidas que solo estan disponibles en branches protegidas y variables escalonadas que requieren un paso de aprobacion manual antes de estar disponibles.

### Limitaciones y riesjos

Las variables de entorno presentan varias limitaciones fundamentales. En primer lugar, son de solo lectura y de tamano limitado: no estan disenadas para manejar certificados completos o archivos grandes. En segundo lugar, una vez inyectadas en el proceso, cualquier script dentro del job puede acceder a todas las variables de entorno, incluyendo aquellas que no necesita. No existe un mecanismo nativo de control de acceso granular a nivel de variable.

El riesgo mas critico es la exposicion en logs. Cualquier comando que imprima el entorno completo (como `env` o `printenv`) revelara todas las variables secretas. Los frameworks de testing como Jest o pytest pueden capturar y registrar el entorno completo en sus reportes de error. Herramientas de build como Gradle o Maven pueden incluir variables de entorno en sus logs de depuracion.

> Un secreto inyectado como variable de entorno esta disponible para TODO el proceso del job, no solo para el script que lo necesita. Si una accion de terceros o un plugin ejecuta un script que recopila el entorno, obtendra todos los secretos. Minimiza el numero de secretos inyectados a los estrictamente necesarios.

## Archivos temporales de secretos

Algunos sistemas CI/CD y herramientas de despliegue esperan encontrar secretos en archivos en lugar de variables de entorno. Esto ocurre frecuentemente con Kubernetes (que espera un kubeconfig), con herramientas de infraestructura como Terraform (que busca credenciales en archivos), y con aplicaciones que leen archivos de configuracion como `.env` o `credentials.json`.

### Patron de archivo temporal

El patron consiste en crear un archivo temporal en el job del pipeline, escribir el secreto en el, ejecutar la herramienta que necesita el archivo, y eliminar el archivo al finalizar. Esto se logra tipicamente con un bloque `before_script` y `after_script` en GitLab CI, o con steps de `run` en GitHub Actions que incluyen cleanup.

La creacion del archivo debe garantizar permisos restrictivos (solo lectura para el usuario del job), un path predecible que la herramienta esperara y limpieza garantizada incluso si el job falla. La limpieza es especialmente critica: si el job se interrumpe por timeout, error o cancelacion, el archivo temporal puede persistir en el sistema de ejecucion, donde otro job podria encontrarlo.

### Riesgos en entornos compartidos

En runners compartidos (como los que ofrecen GitHub Actions o GitLab.com), los archivos temporales pueden persistir en el disco entre ejecuciones si el runner no esta configurado con contenedores efimeros. Un runner que ejecuta multiples jobs en el mismo host sin aislamiento puede exponer archivos temporales de un job a otro. Por esta razon, los runners compartidos deben ejecutarse en entornos efimeros que se destruyen y reconstruyen completamente entre cada job.

Los sistemas de ejecucion auto-hospedados requieren configuracion adicional para garantizar este aislamiento. Cada job debe ejecutarse en un contenedor nuevo, una maquina virtual efimera o un filesystem con capa writeable desechada al finalizar. Sin estas garantias, los archivos temporales de secretos se convierten en un vector de ataque entre usuarios del mismo runner.

## OIDC: autenticacion sin secretos estaticos

OIDC (OpenID Connect) en CI/CD es un enfoque que elimina la necesidad de secretos estaticos de larga duracion al utilizar tokens de identidad de corta vida emitidos por el proveedor de CI/CD y validados directamente por el proveedor de nube o servicio destino.

### Como funciona

El flujo de OIDC para CI/CD se basa en la confianza bidireccional entre el proveedor de CI/CD y el proveedor de servicios. El proveedor de CI/CD (GitHub Actions, GitLab CI, etc.) genera un JWT (JSON Web Token) firmado que contiene informacion sobre la ejecucion: el repositorio, la branch, el commit, el workflow que se ejecuto y el timestamp. El proveedor de servicios (AWS, Azure, GCP, Vault) verifica la firma del token, comprueba que el emisor es el proveedor de CI/CD de confianza y evalua las reclamaciones (claims) del token contra una politica configurada.

Por ejemplo, en AWS se configura un "identity provider" en IAM que reconoce los tokens firmados por GitHub Actions, y luego se crea un "role" con una politica que especifica que solo puede asumirse desde un repositorio y branch concretos. Cuando el pipeline necesita acceder a AWS, solicita un token OIDC, lo intercambia por credenciales temporales de AWS STS y utiliza esas credenciales durante la ejecucion. Al terminar el job, las credenciales expiran automaticamente.

### Ventajas sobre secretos estaticos

La eliminacion de secretos estaticos es la ventaja mas significativa. No hay credenciales que robar, filtrar o rotar porque las credenciales reales solo existen durante los minutos que dura la ejecucion del job y se generan on-demand. Esto elimina por completo el problema de secretos hardcodeados en la configuracion del pipeline y reduce drasticamente la superficie de ataque.

OIDC tambien proporciona auditoria detallada porque cada token contiene informacion rica sobre el contexto de ejecucion. Los logs del servicio destino muestran exactamente que repositorio, commit y workflow generaron cada peticion, lo que facilita la trazabilidad y la respuesta a incidentes.

> OIDC no es una solucion completa de gestion de secretos: las aplicaciones que se despliegan todavia necesitan sus propios secretos (credenciales de base de datos, API keys de terceros). OIDC resuelve el problema del acceso del pipeline a la infraestructura, no el problema de los secretos que la aplicacion consume.

## Auditoria de acceso a secretos

La auditoria es la capacidad de registrar, rastrear y analizar todos los accesos a secretos para detectar usos anomales, cumplir con requisitos de cumplimiento normativo y proporcionar evidencia forense en caso de incidente.

### Que se debe registrar

Cada acceso a un secreto deberia generar un registro que incluya: quien accedio (identidad del solicitante), que secreto pidio (ruta o nombre del recurso), cuando (timestamp preciso), desde donde (IP o identidad del servicio), que operacion realizo (lectura, escritura, borrado) y si la operacion fue exitosa o fallo.

Este nivel de detalle permite detectar patrones sospechosos como accesos fuera del horario laboral, lecturas repetidas del mismo secreto (posible exfiltracion), accesos desde servicios que normalmente no necesitan ese secreto, o intentos de acceso fallidos que podrian indicar un escaneo de permisos.

### Herramientas de auditoria

Los gestores de secretos como Vault y los servicios de nube proporcionan logs de auditoria nativos. Vault genera registros detallados para cada operacion en su "audit device", que se puede configurar para enviar eventos a sistemas de logging centralizado como Splunk, ELK o Datadog. Los servicios de nube como AWS CloudTrail o Azure Monitor registran las llamadas a las API de gestion de secretos.

En entornos de CI/CD, la auditoria implica registrar que secretos se inyectaron en cada ejecucion de pipeline (sin revelar los valores), que scripts accedieron a ellos, y que operaciones realizaron. GitHub Actions, GitLab CI y Jenkins mantienen historiales de ejecucion que muestran que variables estuvieron disponibles, pero el nivel de detalle sobre el uso efectivo de los secretos dentro del job es limitado.

### Retencion y cumplimiento

Los registros de auditoria de secretos deben retenerse durante periodos que cumplan con los requisitos normativos aplicables. Sectores regulados como la banca (PCI DSS), la salud (HIPAA) o el sector publico (GDPR) especifican periodos minimos de retencion que van desde un ano hasta varios anos. Los registros deben almacenarse de forma inmutable y en un sistema separado del que almacena los propios secretos, para que un compromiso del sistema de secretos no permita borrar la evidencia del acceso.

La combinacion de estas practicas crea una defensa en profundidad para los secretos en CI/CD:OIDC elimina los secretos estaticos, las variables de entorno con proteccion de logs reducen la exposicion, los archivos temporales con aislamiento de runners limitan la persistencia, y la auditoria completa la visibilidad necesaria para detectar y responder a incidentes que las demas capas no pudieron prevenir.

---

