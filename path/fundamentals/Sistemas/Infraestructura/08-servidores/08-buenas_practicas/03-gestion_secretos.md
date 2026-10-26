# Gestion de Secretos

Los secretos son las piezas mas sensibles de cualquier infraestructura: claves de acceso a bases de datos, tokens de autenticacion, certificados TLS, credenciales de servicios en la nube y contrasenas de administracion. Su gestion inadecuada es una de las causas mas frecuentes de brechas de seguridad, y paradójicamente, una de las areas donde mas atajos se toman por comodidad o prisa.

## Por que los secretos en codigo son una bomba de relojeria

Almacenar secretos en repositorios de codigo fuente, archivos de configuracion versionados o scripts compartidos es una practica extremadamente peligrosa por varias razones que se refuerzan mutuamente.

### Historial persistente

Los sistemas de control de versiones como Git conservan todo el historial de cambios. Si una clave de acceso se agrega accidentalmente a un repositorio y luego se elimina en el siguiente commit, la clave sigue existiendo en el historial. Cualquier persona con acceso al repositorio puede recuperarla examinando los commits anteriores. Incluso hacer force-push para reescribir el historial no es garantia absoluta, porque otros clones del repositorio pueden contener la version comprometida.

### Difusion incontrolada

Los repositorios de codigo tienden a compartirse, clonarse y duplicarse. Un secreto en un repositorio privado puede acabar en copias locales de desarrolladores, en repositorios de forks, en copias de seguridad, en sistemas de integracion continua y en caches locales. Cada una de esas copias es un punto de exposicion potencial. Una vez que un secreto sale del control directo del equipo de seguridad, recuperarlo es practicamente imposible.

### Detectores automaticos

Existen herramientas como GitLeaks, TruffleHog o las propias funciones de escaneo de GitHub que escanean repositorios en busca de patrones que coincidan con claves de API, tokens y contrasenas. Estos detectores son utilizados tanto por atacantes que buscan secretos expuestos en repositorios publicos como por equipos de seguridad internos. Un secreto en un repositorio sera encontrado, la cuestion es cuando y por quien.

### Escalado del dano

Cuando un secreto se ve comprometido, el dano potencial depende de a que da acceso. Si se trata de la clave de acceso a un servicio de nube, un atacante podria provisionar recursos a costa de la victima, acceder a datos almacenados o utilizar el servicio como punto de pivot para atacar otros sistemas. Si se trata de una clave de base de datos, el dano puede incluir la exfiltracion de datos personales de usuarios, con las consecuencias legales y reputacionales que eso conlleva.

## Herramientas de gestion de secretos

### HashiCorp Vault

Vault es un sistema de gestion de secretos de uso general que almacena, accede y controla secretos de forma centralizada. Su modelo de seguridad se basa en el principio de acceso dinamico: en lugar de distribuir secretos estaticos a los servicios que los necesitan, Vault genera credenciales temporales con una vida util limitada y las revoca automaticamente cuando expiran.

Vault organiza los secretos en rutas logicas y aplica politicas de acceso granulares que determinan que entidades pueden acceder a que secretos y bajo que condiciones. Cada acceso queda registrado en un log de auditoria, proporcionando trazabilidad completa de quien accedio a que secreto y cuando.

Una caracteristica especialmente relevante es la capacidad de generar credenciales dinamicas para bases de datos. En lugar de configurar una contrasena fija para conectarse a MySQL o PostgreSQL, el servicio solicita a Vault credenciales temporales que se crean bajo demanda y se revocan automaticamente despues de un periodo configurado. Esto elimina el problema de las contrasenas estaticas que nunca se rotan.

### AWS Secrets Manager

Secrets Manager es el servicio de gestion de secretos de Amazon Web Services, integrado nativamente con el resto de servicios de la plataforma. Permite almacenar secretos, rotarlos automaticamente (por ejemplo, rotar las credenciales de una base de RDS periodicamente) y acceder a ellos desde servicios Lambda, ECS o EC2 mediante permisos IAM.

Su principal ventaja es la integracion con el ecosistema AWS: un servicio que necesita acceder a un secreto no necesita una configuracion adicional compleja, basta con configurar los permisos IAM correspondientes. La rotacion automatica se configura mediante una funcion Lambda que se encarga de generar nuevas credenciales y actualizarlas en los servicios afectados.

### Variables de entorno

En entornos menos complejos, las variables de entorno representan una alternativa simple pero efectiva. Los secretos se inyectan en el proceso a traves de variables de entorno en lugar de硬编码 en archivos de configuracion. Esta tecnica es especialmente util con contenedores Docker y plataformas de orquestacion como Kubernetes, donde las variables de entorno se pueden gestionar a traves de secrets nativos del orquestador.

La principal limitacion de las variables de entorno es que no ofrecen rotacion automatica, auditoria centralizada ni politicas de acceso granulares. Son adecuadas para entornos de desarrollo o aplicaciones simples, pero insuficientes para infraestructura compleja que requiere cumplimiento normativo.

## Principios de gestion de secretos

### Nunca en texto plano

Esta regla es absoluta e incondicional. Los secretos no deben aparecer nunca en archivos de configuracion sin cifrar, en scripts versionados, en ficheros de log, en correos electronicos o en cualquier medio que no este especificamente disenado para almacenar informacion sensible. No existen excepciones justificadas a esta regla.

### Rotacion periodica

Los secretos deben rotarse periodicamente, independientemente de si se sospecha que han sido comprometidos. La rotacion limita la ventana de exposicion: si un secreto se ve comprometido, solo es util para el atacante durante el periodo hasta la siguiente rotacion. Las herramientas modernas de gestion de secretos permiten automatizar este proceso, eliminando la carga operativa de la rotacion manual.

### Acceso最小imo

Cada servicio o usuario solo debe tener acceso a los secretos que necesita estrictamente para su funcion. Un servidor web no necesita acceso a las credenciales de la base de datos de analitica. Un servicio de log no necesita las claves de cifrado de datos de usuario. La granularidad en el acceso a secretos limita el dano en caso de compromiso de un componente.

### Registro de accesos

Todo acceso a un secreto debe quedar registrado: quien accedio, cuando, desde donde y para que proposito. Los registros de acceso son esenciales para la deteccion de usos anomalous y para la investigacion post-incidente. Si un secreto se ve comprometido, el registro de accesos permite determinar que servicios podrian estar afectados.

