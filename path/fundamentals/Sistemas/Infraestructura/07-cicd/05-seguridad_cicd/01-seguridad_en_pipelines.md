# Seguridad en Pipelines CI/CD

Un pipeline de integracion y despliegue continuo concentra un poder excepcional sobre el ciclo de vida del software: compila codigo, ejecuta pruebas, genera artefactos y despliega en entornos de produccion. Ese mismo poder lo convierte en un objetivo privilegiado para un atacante. Comprometer un pipeline no requiere explotar una vulnerabilidad en la aplicacion final; basta con obtener credenciales que el propio pipeline ya posee para acceder a repositorios, registries, servidores de despliegue o bases de datos de produccion. La seguridad de los pipelines CI/CD no es un complemento opcional, sino una condicion necesaria para que la automatizacion sea una ventaja y no un vector de ataque amplificado.

## Por que los pipelines son un vector de ataque prioritario

Un pipeline de CI/CD opera con privilegios que ningun desarrollador individual deberia tener de forma permanente. En el transcurso de una ejecucion tipica, el pipeline necesita leer codigo fuente, descargar dependencias, compilar artefactos, publicar imagenes en un registry, conectarse a servidores de despliegue via SSH o API, y en ocasiones ejecutar migraciones de bases de datos. Cada uno de esos pasos requiere credenciales: tokens de acceso, contrasenas de servicios, llaves SSH, certificados TLS o claves de cifrado. Si un atacante logra inyectar codigo malicioso en el pipeline — ya sea a traves de un commit directo, una dependencia comprometida, o un plugin malicioso — obtiene acceso implicito a todos esos recursos sin necesidad de vulnerar ningun sistema externo.

El problema se agrava porque los pipelines suelen ejecutarse en entornos donde las politicas de seguridad son mas laxas que en produccion. Los runners de CI son maquinas efimeras que ejecutan codigo de terceros (acciones de GitHub, plugins de Jenkins, stages de GitLab CI) sin la revision exhaustiva que se aplicaria a un despliegue en produccion. Ademas, las credenciales almacenadas en el pipeline son estaticas y de larga duracion, lo que amplifica la ventana de explotacion en caso de compromiso.

## Gestion de secretos en pipelines

### El problema de los secretos estaticos

La forma mas comun — y mas insegura — de proporcionar credenciales a un pipeline es codificarlas directamente en los archivos de configuracion o en variables de entorno del sistema de integracion. Tokens como `GITHUB_TOKEN=ghp_abc123...` en un archivo `.gitlab-ci.yml` o un `Jenkinsfile` commiteado al repositorio constituyen una exposicion inmediata: cualquier persona con acceso al repositorio (o cualquier fork publico) puede extraerlos. Incluso si se eliminan en un commit posterior, permanecen en el historial de git.

### Enfoques para gestionar secretos de forma segura

La gestion moderna de secretos en pipelines se articula en torno a varios principios fundamentales:

**Variables cifradas del propio sistema CI.** Herramientas como GitLab CI, GitHub Actions y Jenkins ofrecen mecanismos nativos para almacenar secretos cifrados en reposo que solo se descifran durante la ejecucion del pipeline y nunca se exponen en logs. GitLab utiliza un cifrado asimetrico donde el sistema CI tiene la clave privada necesaria para descifrar las variables marcadas como "masked" y "protected". GitHub Actions almacena los secrets en un vault interno y los inyecta como variables de entorno, con la particularidad de que si un secreto aparece en la salida de un comando, el sistema lo enmascara automaticamente con `***`.

**External Secret Managers.** Para organizaciones con requisitos de cumplimiento normativo o necesidades de rotacion avanzada, se emplean soluciones externas como HashiCorp Vault, AWS Secrets Manager o Azure Key Vault. En este modelo, el pipeline no almacena secretos; en su lugar, ejecuta un agente que solicita credenciales de forma dinamica al momento de necesitarlas, con un TTL corto y auditoria completa de cada acceso. Vault, por ejemplo, puede generar credenciales efimeras de base de datos que se revocan automaticamente una vez finalizada la ejecucion del pipeline.

**OIDC y tokens de corta duracion.** La tendencia mas reciente elimina la necesidad de almacenar secretos a largo plazo mediante protocolos de federacion de identidad. GitHub Actions, por ejemplo, puede emitir un token OIDC que un proveedor externo (AWS, GCP, Vault) valida y utiliza para emitir credenciales temporales. El pipeline nunca posee una clave estatica; simplemente se autentica ante el proveedor con su identidad verificada y recibe credenciales que expiran en minutos.

### Principios de minimo privilegio

Mas alla del mecanismo de almacenamiento, la seguridad de secretos exige que cada pipeline o job solo reciba las credenciales estrictamente necesarias para su tarea. Un job de compilacion no necesita credenciales de despliegue; un job de pruebas no necesita acceso al registry de imagenes. La segmentacion de secretos por job reduce significativamente el impacto de un compromiso parcial del pipeline.

> Los secretos nunca deben imprimirse en logs. Ademas de las protecciones nativas del sistema CI, los pipelines deben ejecutarse con opciones como `set +x` para desactivar el echo de comandos, y los plugins de logging deben configurarse para filtrar patrones conocidos de tokens y contrasenas.

## Escaneo de vulnerabilidades

### SAST (Static Application Security Testing)

El escaneo estatico analiza el codigo fuente sin ejecutarlo, buscando patrones que indiquen vulnerabilidades potenciales: inyeccion de SQL, cross-site scripting, uso de funciones criptograficas debiles, manejo inadecuado de excepciones o exposicion de datos sensibles. Herramientas como SonarQube, Semgrep, CodeQL o Bandit (para Python) se integran en etapas tempranas del pipeline, idealmente despues del checkout del codigo y antes de la compilacion.

La ventaja del SAST es que detecta problemas cuando son mas baratos de corregir: en fase de desarrollo, antes de que el codigo llegue a pruebas o despliegue. Sin embargo, el SAST genera falsos positivos significativos y no puede detectar vulnerabilidades que surgen de la interaccion entre componentes en tiempo de ejecucion. Por eso es complementario, no sustitutivo, de otras tecnicas.

### DAST (Dynamic Application Security Testing)

A diferencia del SAST, el escaneo dinamico ataca la aplicacion en ejecucion desde el exterior, simulando lo que haria un atacante real. Herramientas como OWASP ZAP, Burp Suite o Nikto envian peticiones HTTP a la aplicacion desplegada en un entorno de pruebas, buscando vulnerabilidades como Cross-Site Scripting, inyeccion de comandos, problemas de autenticacion o configuraciones inseguras de cookies y headers.

El DAST tiene la ventaja de detectar vulnerabilidades reales que solo se manifiestan en runtime, pero requiere una aplicacion desplegada y funcional, lo que lo situa mas tarde en el pipeline (generalmente despues del despliegue a un entorno de staging). Ademas, la cobertura depende criticamente de la calidad de los casos de prueba que se proporcionan a la herramienta.

### SCA (Software Composition Analysis)

El SCA analiza las dependencias de terceros del proyecto, comparandolas contra bases de datos de vulnerabilidades conocidas como NVD (National Vulnerability Database) o GitHub Advisory Database. Herramientas como Snyk, Dependabot, OWASP Dependency-Check o Trivy identifican cuando una libreria utilizada en el proyecto tiene CVEs publicados y proporcionan información sobre la severidad, el alcance y las versiones afectadas.

El SCA es especialmente critico en el ecosistema actual, donde una aplicacion tipica depende de cientos o miles de paquetes de terceros. Una vulnerabilidad en una dependencia transitiva — una libreria que no se importa directamente pero que es requerida por otra — puede pasar desapercibida durante meses sin un escaneo automatizado.

### Estrategia de escaneo integrada

La implementacion efectiva del escaneo de vulnerabilidades combina los tres enfoques en puntos strategicos del pipeline:

- **Pre-commit / Pre-push:** SAST ligero como hook local para retroalimentacion inmediata al desarrollador.
- **Post-commit:** SAST completo y SCA para analizar el codigo integrado y sus dependencias.
- **Post-build:** SAST sobre el artefacto compilado y escaneo de imagenes Docker.
- **Post-deploy (staging):** DAST contra la aplicacion desplegada en un entorno de pruebas.

Cada escaneo debe configurarse con un umbral de severidad minimo para bloquear el pipeline. Un hallazgo de severidad critica debe impedir que el codigo avance; los hallazgos de severidad media o baja pueden registrarse para corregirse sin detener el proceso, segun la politica de la organizacion.

## Escaneo de imagenes Docker

Las imagenes Docker representan un vector de ataque particular porque encapsulan no solo el codigo de la aplicacion, sino tambien el sistema operativo, las librerias del sistema, herramientas instaladas y la configuracion. Una imagen basada en `ubuntu:22.04` hereda todas las vulnerabilidades conocidas de esa version base, y una mala practica como instalar herramientas de compilacion en la imagen final expande innecesariamente la superficie de ataque.

El escaneo de imagenes Docker se realiza con herramientas como Trivy, Grype, Snyk Container o Docker Scout, que analizan las capas de la imagen, identifican los paquetes instalados (tanto del sistema operativo como del gestor de dependencias de la aplicacion) y los cruzan contra bases de datos de vulnerabilidades. Un escaneo tipico busca CVEs en el SO base, en las dependencias de la aplicacion y en la configuracion de la imagen (usuarios root, puertos expuestos, archivos con permisos excesivos).

> Trivy se ha consolidado como herramienta de referencia para el escaneo de imagenes porque soporta multiples formatos (Docker, OCI, tarballs), detecta problemas de configuracion (misconfiguration), secrets expuestos en las capas, y se integra facilmente en pipelines con un unico comando. Su capacidad de escaneo de SBOM lo convierte en una herramienta unificada para cumplir multiples requisitos de seguridad.

Las mejores practicas para el escaneo de imagenes incluyen:

- Escanear en el pipeline antes de hacer push al registry, para evitar propagar imagenes vulnerables.
- Configurar politicas de admision en el cluster Kubernetes que rechacen imagenes sin escanear o con vulnerabilidades criticas.
- Utilizar imagenes base minimales (Alpine, Distroless, Scratch) para reducir la superficie de ataque desde el diseño.
- Firmar las imagenes con Cosign o Notary para garantizar que la imagen desplegada es exactamente la que fue aprobada por el escaneo.

## Firma y verificacion de integridad de artefactos

### El problema de la cadena de confianza

Cuando un pipeline genera un artefacto (un binario, una imagen Docker, un paquete de librerias), el consumidor final — ya sea un servidor de despliegue, un desarrollador o un sistema de orquestacion — necesita garantizar dos propiedades fundamentales: que el artefacto no ha sido alterado desde su creacion (integridad) y que efectivamente fue generado por el pipeline legitimno (autenticidad). Sin mecanismos de verificacion, un atacante podria intercambiar un artefacto legitimo por uno malicioso en cualquier punto de la cadena de distribucion.

### Cosign y el proyecto Sigstore

Cosign, parte del ecosistema Sigstore, permite firmar imagenes de contenedores, artifacts OCI y manifests de Kubernetes utilizando claves criptograficas o identidades OIDC. En el modelo sin claves (keyless), el pipeline se autentica ante Fulcio (un CA efimero) utilizando su identidad OIDC del proveedor CI (el claim `sub` del token OIDC), recibe un certificado temporal, y utiliza ese certificado para firmar el artefacto. La firma se almacena junto al artefacto en el registry como una anotacion OCI.

La verificacion es asimetrica: el consumidor consulta Rekor (un log de transparencia inmutable) para obtener la firma y el certificado, verifica que el certificado fue emitido por una CA de confianza, y valida que la identidad OIDC coincide con la esperada (por ejemplo, que la firma fue emitida por el repo `mi-organizacion/backend` en GitHub Actions). Todo el proceso es auditable y no requiere gestion manual de claves.

### SLSA y la integridad de la cadena de suministro

El framework SLSA (discutido con mayor detalle en el siguiente apunte) define niveles de integridad que incluyen la generacion automatica de provenance: un artefacto firmado no solo prueba que no fue alterado, sino que fue construido por un pipeline especifico a partir de un commit especifico del codigo fuente. Esta trazabilidad completa cierra el ciclo de confianza desde el codigo hasta el artefacto desplegado.

### Attestations y politicas de admision

Mas alla de la firma, las attestations son declaraciones criptograficas adjuntas a un artefacto que certifican propiedades sobre el: que paso por un escaneo de vulnerabilidades sin hallazgos criticos, que fue construido en un entorno reproducible, que el codigo fuente es el que indica el commit hash. Herramientas como Kyverno o OPA Gatekeeper en Kubernetes pueden configurarse para rechazar cualquier contenedor que no posea attestations validas firmadas por un identity de confianza.

> El principio subyacente es la confianza zero: no se confia en un artefacto simplemente porque proviene de un pipeline conocido, sino porque puede demostrar criptograficamente su origen, su integridad y que cumplio con los controles de seguridad definidos.

