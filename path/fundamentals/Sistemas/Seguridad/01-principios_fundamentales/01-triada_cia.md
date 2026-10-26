# Triada CIA

La seguridad informatica no es un concepto binario donde algo esta "seguro" o "no esta seguro", sino un espectro de compromisos entre propiedades fundamentales que determinan el nivel de proteccion de un sistema. La triada CIA (Confidentiality, Integrity, Availability) constituye el marco conceptual mas antiguo y universal en seguridad, estableciendo las tres pilares sobre los cuales se construye cualquier politica de proteccion. Comprender estas propiedades no es solo un ejercicio teorico: cada decision tecnica que tomamos, desde configurar un firewall hasta disenar una base de datos, implica balancear estas tres dimensiones que frecuentemente entran en conflicto entre si.

## Confidencialidad

La confidencialidad garantiza que la informacion solo sea accesible para entidades autorizadas. No se trata simplemente de "ocultar" datos, sino de establecer mecanismos que dificulten o impidan el acceso no autorizado. En infraestructura, esto se manifiesta en multiples capas: contrasenas robustas, cifrado de datos en reposo y en transito, segmentacion de redes, listas de control de acceso (ACL) y politicas de privilegio minimo.

El ejemplo mas cotidiano es el cifrado TLS: cuando accedemos a un banco por HTTPS, la confidencialidad asegura que un atacante que intercepte el trafico no pueda leer las credenciales. En el contexto de servidores, la confidencialidad implica que un usuario con privilegios de lectura en un directorio no pueda acceder a archivos de configuracion de otros servicios, o que un contenedor Docker no pueda inspeccionar el trafico de otro contenedor en la misma maquina.

Un matiz importante es que la confidencialidad absoluta es practicamente imposible y frecuentemente contraproducente. Si ciframos cada byte de datos con algoritmos de 4096 bits y mantenemos las claves en un deposito fisico separado, la informacion sera extremadamente confidencial pero inutilizable en la practica. El equilibrio consiste en determinar el nivel de proteccion apropiado segun la sensibilidad de los datos.

## Integridad

La integridad asegura que los datos no sean alterados de forma no autorizada, ya sea durante su almacenamiento o durante su transmision. Esto abarca tanto la prevencion de modificaciones maliciosas como la deteccion de errores accidentales. Un sistema con alta integridad puede garantizar que un archivo de configuracion no ha sido modificado desde su ultima revision autorizada, o que una transaccion bancaria ha llegado al destino exactamente igual que cuando fue enviada.

En infraestructura, la integridad se implementa mediante checksums (SHA-256, por ejemplo) que permiten verificar si un archivo ha sido modificado, firmas digitales que autentican el origen de paquetes de software, y herramientas como AIDE o Tripwire que monitorean cambios en archivos criticos del sistema. Los repositorios de paquetes utilizan firmas GPG precisamente para garantizar que el software que instalamos es exactamente el que el autor publico, sin modificaciones intermedias.

La integridad y la confidencialidad a menudo se refuerzan mutuamente, pero tambien pueden generar tensiones. Un sistema que registra cada intento de acceso (para preservar la integridad del registro) consume recursos que podrian afectar la disponibilidad. Ademas, verificar integridad constantemente puede introducir latencia en operaciones de lectura/escritura, creando un trade-off directo entre garantizar que los datos no han sido modificados y mantener el rendimiento del sistema.

## Disponibilidad

La disponibilidad garantiza que los sistemas y datos esten accesibles cuando los usuarios autorizados los necesitan. No es simplemente una cuestion de mantener un servidor encendido, sino de asegurar que todos los componentes criticos funcionen dentro de los niveles de servicio acordados. Esto incluye redundancia de hardware, balanceo de carga, planes de recuperacion ante desastres y proteccion contra ataques de denegacion de servicio (DDoS).

Un ejemplo concreto es una tienda en linea: si el servidor web cae durante el periodo de Navidad, la falta de disponibilidad genera perdidas economicas directas mas alla de cualquier consideracion de confidencialidad o integridad. En infraestructura critica como hospitales o plantas de energia, la disponibilidad puede ser literalmente una cuestion de vidas humanas, lo que justifica inversiones significativas en redundancia y tolerancia a fallos.

La disponibilidad entra en conflicto directo con la confidencialidad en multiples escenarios. Un sistema que requiere autenticacion multi-factor con tokens fisicos ofrece alta confidencialidad, pero si el usuario pierde su token o el servicio de autenticacion cae, la informacion legittima se vuelve inaccesible. Del mismo modo, los ataques DDoS explotan este tension: el atacante no necesita romper la confidencialidad ni la integridad, solo necesitan destruir la disponibilidad. Las contramedidas contra DDoS (como CDN, rate limiting y scrubbing centers) representan inversiones significativas dedicadas exclusivamente a esta propiedad.

## Interacciones y trade-offs

La triada CIA no es una lista de verificacion donde se pueden maximizar las tres propiedades simultaneamente. Cada decision de seguridad implica un compromiso. El cifrado extremo protege la confidencialidad pero puede degradar la disponibilidad si los procesos de descifrado son lentos o si las claves se pierden. Los sistemas de backup frecuentes protegen la disponibilidad pero crean superficies de ataque para la confidencialidad, ya que cada copia es un punto potencial de fuga.

La clave esta en evaluar el contexto. Un servidor DNS publico prioriza la disponibilidad sobre la confidencialidad (los registros DNS son informacion publica por naturaleza). Un servidor de registros medicos prioriza la confidencialidad y la integridad sobre la disponibilidad en ciertos escenarios. Un sistema de procesamiento de pagos necesita equilibrar las tres propiedades de forma equitativa, ya que la falta de cualquiera de ellas tiene consecuencias graves.

---

