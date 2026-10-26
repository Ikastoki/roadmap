# PKI y Certificados

La criptografia asimetrica resuelve el problema de comunicacion segura entre dos partes que no se conocen previamente, pero presenta una limitacion critica: como verifica un cliente que la clave publica que recibe realmente pertenece al servidor que dice ser y no a un atacante que intercepta la comunicacion? La infraestructura de claves publicas (PKI) emerge como la respuesta a esta pregunta, estableciendo un marco de confianza jerarquico que vincula identidades con claves publicas mediante la autoridad de certificacion.

## Que es una PKI

Una infraestructura de claves publicas es un conjunto de politicas, procedimientos, hardware y software necesarios para gestionar el ciclo de vida de los certificados digitales. La PKI no es simplemente una coleccion de certificados, sino un ecosistema completo que incluye la emision, distribucion, renovacion, revocacion y verificacion de certificados. Su proposito fundamental es resolver el problema de confianza en entornos digitales donde las partes comunicantes no pueden verificarse presencialmente.

Una PKI eficaz debe soportar millones de certificados simultaneos, proporcionar mecanismos de revocacion en tiempo real, mantener registros auditables de todas las operaciones y ser resistente a compromisos de seguridad. Organizaciones gubernamentales, bancos y proveedores de servicios cloud operan PKI propias, mientras que la mayoria de organizaciones dependen de PKI publicas gestionadas por autoridades de certificacion externas.

> Una PKI bien disenada es transparente para el usuario final. Cuando visitas un sitio web con HTTPS, la validacion del certificado ocurre automaticamente en milisegundos, sin intervention manual.

## Certificados X.509

Los certificados X.509 son el estandar internacional para la estructura de certificados de clave publica. Un certificado X.509 contiene informacion esencial que vincula una identidad (nombre de dominio, organizacion, etc.) con una clave publica especifica, todo ello firmado digitalmente por una autoridad de certificacion.

Los campos principales de un certificado X.509 incluyen:

- **Version**: especifica la version del estandar X.509 (actualmente v3)
- **Numero de serie**: identificador unico emitido por la CA
- **Algoritmo de firma**: algoritmo utilizado por la CA para firmar el certificado
- **Emisor (Issuer)**: nombre distinguido de la CA que emite el certificado
- **Periodo de validez**: fechas de inicio y fin de validez (Not Before/Not After)
- **Sujeto (Subject)**: nombre distinguido de la entidad propietaria del certificado
- **Clave publica**: la clave publica del sujeto
- **Extensiones**: campos opcionales como Subject Alternative Names, Basic Constraints, Key Usage

Las extensiones son particularmente importantes en certificados modernos. Subject Alternative Names (SAN) permiten que un certificado sea valido para multiples dominios o subdominios. Basic Constraints determina si un certificado puede firmar otros certificados (CA=true) o es solo un certificado de extremo (CA=false). Key Usage restringe los usos permitidos del certificado, como cifrado, firma o firma de certificados.

## Autoridades de Certificacion

Una autoridad de certificacion es la entidad de confianza responsable de emitir, gestionar y revocar certificados digitales. La CA actua como notario digital: verifica la identidad del solicitante antes de emitir un certificado y garantiza que la informacion contenida en el certificado es correcta. La confianza en todo el sistema depende de la integridad y seguridad de las CA.

Existen diferentes tipos de CA con distintos niveles de rigurosidad en la verificacion:

- **CA raiz (Root CA)**: la autoridad de最高 confianza en la jerarquia. Sus certificados auto-firmados se distribuyen preinstalados en sistemas operativos y navegadores. La compromision de una CA raiz compromete toda la PKI dependiente.
- **CA intermedia (Intermediate CA)**: opera bajo la autoridad de una CA raiz para emitir certificados de终端. Proporcionan una capa adicional de seguridad: si una CA intermedia es comprometida, solo afecta a los certificados que ella emite, no a toda la jerarquia.
- **CA de subordinada**: puede delegar autoridad a otras CA para gestionar certificados en dominios especificos o regiones geograficas.

> Las CA raiz mantienen sus claves privadas en Hardware Security Modules (HSM) con multiples capas de proteccion fisica y logica. La compromision de una CA raiz es uno de los peores escenarios de seguridad en PKI, ya que permite a un atacante generar certificados fraudulentos que serian aceptados por cualquier sistema.

## Cadena de Confianza

La cadena de confianza es el mecanismo que conecta un certificado de终端 con una CA raiz confiable. Cuando un navegador recibe un certificado TLS, no solo valida ese certificado especifico, sino toda la cadena hasta llegar a una CA raiz que reconoce como confiable. Este proceso se conoce como construccion y validacion de la cadena de certificados.

El proceso funciona de la siguiente manera:

1. El servidor presenta su certificado de终端 junto con todos los certificados intermedios necesarios
2. El cliente verifica que el certificado de终端 esta firmado por uno de los certificados intermedios proporcionados
3. Verifica que el certificado intermedio esta firmado por una CA raiz en su almacen de confianza
4. Verifica que ningun certificado en la cadena este revocado (mediante CRL o OCSP)
5. Verifica que la fecha actual este dentro del periodo de validez de todos los certificados
6. Verifica que los campos Basic Constraints y Key Usage sean consistentes con el rol de cada certificado

Si cualquier paso falla, el cliente rechaza la conexion y muestra un aviso de seguridad. La omision de certificados intermedios es una causa comun de errores de validacion: el servidor debe enviar la cadena completa, no solo su propio certificado.

## Por que Necesitamos CAs

La pregunta natural es: si la criptografia asimetrica ya permite intercambio seguro de claves, por que no simplemente intercambiar claves publicas directamente sin intermediarios? La respuesta radica en el problema de autenticacion. Sin una CA, un cliente no tiene forma de determinar si la clave publica que recibe pertenece realmente al servidor legítimo o a un atacante que intercepta la comunicacion (ataque man-in-the-middle).

Las CA resuelven este problema estableciendo una tercera parte de confianza. La CA garantiza que la clave publica contenida en un certificado realmente pertenece a la entidad nombrada en ese certificado. Este modelo de confianza es similar al sistema bancario: confiamos en el banco no porque conozcamos todos los detalles de su operacion, sino porque existe un marco regulatorio y de auditoria que garantiza su fiabilidad.

Sin embargo, este modelo de confianza centralizado tiene vulnerabilidades. Si una CA es comprometida o actua de forma maliciosa, puede emitir certificados fraudulentos que serian aceptados universalmente. Historialmente, varias CA han sido comprometidas, lo que ha llevado al desarrollo de tecnologias complementarias como Certificate Transparency (CT), que registra publicamente todos los certificados emitidos para deteccion temprana de emisiones fraudulentas, y Certificate Authority Authorization (CAA), que permite a los dominios especificar que CA esta autorizada a emitir certificados para ellos.

---

