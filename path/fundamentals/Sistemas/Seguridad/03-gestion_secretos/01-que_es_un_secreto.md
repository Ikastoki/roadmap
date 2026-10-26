# Que es un secreto

Un secreto en el contexto de infraestructura y seguridad informatica es cualquier pieza de informacion que, si cae en manos equivocadas, permite a un atacante obtener acceso no autorizado a sistemas, datos o servicios. Los secretos constituyen la primera linea de defensa en cualquier arquitectura: controlan quién puede autenticarse, autorizarse y comunicarse de forma segura entre componentes. Cuando un secreto se ve comprometido, toda la cadena de confianza construida a su alrededor se desmorona, independientemente de lo robusto que sea el resto del sistema.

## Tipos de secretos

Los secretos adoptan multiples formas dependiendo de su proposito y contexto de uso. Cada tipo presenta caracteristicas distintas que determinan como debe almacenarse, rotarse y protegerse.

### Contrasenas y credenciales de usuario

Las contrasenas son el tipo de secreto mas conocido. Representan un factor de autenticacion basado en conocimiento que protege cuentas de usuario, bases de datos y servicios. En entornos de servidor, las contrasenas de base de datos, usuarios de servicio o cuentas administrativas son especialmente criticas porque su compromiso puede otorgar acceso privilegiado a datos sensibles o al control completo de un sistema.

Las contrasenas de usuario humano presentan un desafio particular: deben ser recordables para las personas pero dificiles de adivinar para algoritmos. Este equilibrio se ha vuelto cada vez mas dificil a medida que los ataques de fuerza bruta y de diccionario se han vuelto mas sofisticados.

### Tokens de acceso y sesion

Un token es una credencial de corta vida que representa la autorizacion de un usuario o servicio despues de una autenticacion inicial. A diferencia de las contrasenas, los tokens tienen un formato estructurado (como JWT - JSON Web Tokens) que contiene informacion sobre el emisor, el sujeto autorizado, los permisos concedidos y la fecha de expiracion.

Los tokens de sesion permiten a los servidores validar peticiones sin consultar una base de datos centralizada en cada request, ya que toda la informacion necesaria esta codificada dentro del propio token. Sin embargo, esta caracteristica tambien los convierte en un objetivo atractivo: quien posee un token valido tiene acceso inmediato sin necesidad de conocer la contrasena original.

### Claves de API

Las claves de API son tokens de larga duracion que identifican y autentican a una aplicacion o servicio frente a un endpoint externo. Se utilizan comunmente para acceder a servicios en la nube (AWS, Azure, GCP), APIs de terceros (Stripe, Twilio, OpenAI) o servicios internos. A diferencia de los tokens de sesion, las claves de API no suelen expirar automaticamente y, por lo tanto, representan un riesgo mayor si se filtran: un atacante puede usarlas durante un periodo prolongado antes de que se detecte el compromiso.

Las claves de API suelen seguir un formato predecible (por ejemplo, `sk_live_`, `AKIA`, `ghp_`) que facilita tanto su deteccion automatizada por parte de herramientas de escaneo como su busqueda por parte de atacantes que rastrean repositorios publicos en busca de patrones coincidentes.

### Certificados y claves privadas

Los certificados TLS/SSL y sus claves privadas asociadas son secretos criptograficos que permiten la comunicacion segura entre sistemas. Un certificado contiene la clave publica y la identidad del titular, mientras que la clave privada (que nunca debe compartirse) permite descifrar datos cifrados con la clave publica y demostrar la identidad durante el handshake TLS.

La compromision de una clave privada permite a un atacante realizar ataques man-in-the-middle, suplantar servicios y firmar datos con la identidad del titular legitimo. Estos secretos tienen una vida util larga (meses o anos) y su rotacion es mas compleja que la de una contrasena simple, lo que intensifica la necesidad de protegerlos adecuadamente.

## Por que no se deben codificar secretos en el codigo

Hardcodear secretos directamente en el codigo fuente es una de las practicas mas peligrosas en desarrollo de software. Las razones son multiples y se refuerzan mutuamente.

### Visibilidad en repositorios

El codigo fuente se almacena en repositorios de control de versiones como Git. Estos repositorios son compartidos entre multiples desarrolladores y, frecuentemente, se almacenan en plataformas como GitHub, GitLab o Bitbucket donde pueden ser accesibles a un numero amplio de personas. Un secreto commitado accidentalmente queda visible para todos los que tengan acceso al historial del repositorio, incluyendo colaboradores externos, proveedores de CI/CD y, en repositorios publicos, el mundo entero.

Incluso si se elimina un secreto del codigo en un commit posterior, permanece en el historial de Git y puede recuperarse facilmente con un `git log` o `git show`. La unica forma segura de eliminar un secreto de un repositorio es reescribir el historial completa o eliminar el repositorio y crear uno nuevo.

### Difusion en multiples contextos

Un secreto hardcodeado viaja con el codigo a todos los entornos donde se despliega: desarrollo, testing, staging y produccion. Esto significa que una misma clave de API protege todos los entornos con la misma credencial. Si el entorno de desarrollo (menos protegido) es comprometido, el atacante obtiene acceso a produccion. Esta falta de separacion de privilegios entre entornos amplifica exponencialmente el impacto de cualquier compromiso.

### Facilidad de extraccion automatizada

Los atacantes utilizan herramientas automatizadas que escanean continuamente repositorios publicos en busca de patrones que coincidan con secretos conocidos. Herramientas como truffleHog, GitLeaks y GitHub Secret Scanning realizan esta labor de forma sistematica. Un secreto hardcodeado puede ser detectado y explotado en cuestione de minutos despues de hacerse publico.

## Rotacion de secretos

La rotacion de secretos es el proceso de reemplazar un secreto existente por uno nuevo de forma periodica y controlada. Este proceso es fundamental porque reduce la ventana de oportunidad para un atacante: incluso si un secreto se ve comprometido, sera valido solo durante el periodo comprendido entre la ultima rotacion y la proxima.

### Frecuencia recomendada

La frecuencia de rotacion depende del tipo de secreto y del nivel de riesgo asociado. Las contrasenas de servicios criticos deberian rotarse cada 30 a 90 dias, mientras que las claves de API de terceros pueden rotarse trimestral o semestralmente. Los tokens de corta vida (como JWT) se consideran auto-rotantes porque expiran en cuestiones de minutos u horas y se renuevan automaticamente.

No existe una frecuencia unica correcta: lo importante es que la rotacion sea un proceso automatico y no dependa de la accion manual de una persona, porque los humanos olvidan, posponen y cometen errores. Un sistema bien diseniado deberia poder rotar cualquier secreto sin interrumpir el servicio, utilizando un proceso de tipo "dual credential" donde el secreto nuevo se despliega primero y el antiguo se revoca despues.

### Proceso de rotacion seguro

Un proceso de rotacion seguro debe cumplir varias propiedades: atomicidad (el cambio de secreto ocurre de forma indivisible o con un mecanismo de transicion), disponibilidad (el servicio nunca queda sin un secreto valido) y auditoria (cada rotacion queda registrada con timestamp, actor responsable y resultado).

El flujo basico consiste en generar el nuevo secreto, actualizarlo simultaneamente en el almacen de secretos y en todos los consumidores, verificar que el nuevo secreto funciona correctamente, y finalmente revocar o eliminar el antiguo. Este proceso es idealmente automatizado mediante herramientas de gestion de secretos o politicas configuradas en el proveedor correspondiente.

> La rotacion manual de secretos es insostenible a escala. Un sistema que depende de que una persona cambie credenciales periodicamente esta condenado al olvido. La automatizacion no es opcional, es un requisito de seguridad fundamental.

## Secretos como vector de ataque

Los secretos filtrados son uno de los vectores de ataque mas comunes y efectivos en el panorama actual de ciberseguridad. Los informes anuales de incidentes de seguridad (como los de Verizon DBIR o GitGuardian) muestran consistentemente que las credenciales comprometidas estan detras de un porcentaje significativo de brechas de datos.

### Por que son tan efectivos

Un secreto filtrado ofrece a un atacante acceso autenticado y autorizado sin necesidad de explotar vulnerabilidades en el codigo, configurar herramientas de explotacion complejas o superar mecanismos de defensa como firewalls o WAFs. El atacante simplemente utiliza el secreto tal cual lo haria el titular legitimo. Esto dificulta enormemente la deteccion porque las peticiones autenticadas con un secreto valido parecen completamente normales en los logs de acceso.

### Vectores de filtracion mas comunes

Las principales fuentes de filtracion de secretos incluyen repositorios Git con secretos commiteados (el vector mas frecuente), logs de aplicaciones que registran variables de entorno completas, archivos de configuracion subidos a repositorios publicos, contenedores Docker que incluyen secretos en sus capas, y copias de seguridad no cifradas que contienen archivos con credenciales.

El fenomeno del "secreto rotting" (secreto podrido) agrava el problema: secretos que nadie recuerda haber creado, que pertenecen a servicios abandonados pero que siguen activos, y que no se han rotado en anos. Estos secretos fantasma representan un riesgo invisible porque nobody los monitorea ni gestiona.

### Impacto real

Una vez que un atacante obtiene un secreto valido, la progresion del ataque suele seguir un patron predecible: primeramente valida el acceso con el secreto, luego explora que recursos y datos son accesibles, a continuacion escala privilegios buscando otros secretos o credenciales mas elevadas, y finalmente establece persistencia creando nuevos usuarios o credenciales propias. Cada paso es mas dificil de detectar que el anterior porque el atacante utiliza mecanismos de acceso legitimo.

La gestion adecuada de secretos no es simplemente una buena practica de seguridad: es un requisito basico de higiene digital que determina si un sistema es vulnerable a un amplio espectro de ataques de bajo esfuerzo y alto impacto.

---

