# Herramientas de gestion de secretos

La gestion de secretos requiere herramientas especializadas que centralicen el almacenamiento, acceso, rotacion y auditoria de credenciales y datos sensibles. Cada herramienta responde a un contexto diferente: entornos hibridos, nubes especificas, plataformas Kubernetes o proyectos con presupuesto limitado. Elegir la solucion adecuada implica comprender las capacidades, limitaciones y modelo de confianza de cada una.

## Hashicorp Vault

Vault es la herramienta mas completa y flexible para la gestion centralizada de secretos. No es simplemente un almacen clave-valor: es un servidor con una arquitectura modular que soporta multiples metodos de autenticacion, multiples motores de secretos y politicas de acceso granulares.

### Arquitectura y conceptos centrales

Vault funciona como un servidor HTTP que expone una API. Los clientes se autentican mediante un "auth method" (approle, kubernetes, userpass, OIDC, certificados TLS, entre otros) y reciben un token que les concede acceso a rutas especificas del almacen segun las politicas ACL configuradas.

Los "secret engines" son modulos que gestionan un tipo concreto de secreto. El motor `kv` almacena pares clave-valor simples; el motor `database` genera credenciales dinamicas efimeras para bases de datos (MySQL, PostgreSQL, MongoDB); el motor `pki` genera certificados TLS bajo una CA interna; el motor `transit` ofrece cifrado como servicio sin exponer las claves de cifrado. Esta modularidad permite que Vault sea el unico punto de gestion para practicamente cualquier tipo de secreto.

Los "policies" definen que rutas puede leer, escribir o listar cada token. Se expresan en un formato declarativo que permite reglas extremadamente granulares, como permitir a un servicio leer solo un secreto concreto en una ruta especifica durante una ventana horaria determinada.

### Credenciales dinamicas

Una de las capacidades mas poderosas de Vault es la generacion de credenciales dinamicas de corta vida. En lugar de almacenar una contrasena estatica para una base de datos, Vault puede crear un usuario temporal con permisos limitados que expira automaticamente despues de un periodo configurable. Esto elimina por completo el problema de las contrasenas compartidas y la necesidad de rotacion manual.

El ciclo de vida es el siguiente: el cliente solicita credenciales a Vault, Vault genera un usuario temporal en la base de datos usando credenciales root almacenadas de forma segura, el cliente usa esas credenciales y, cuando expira el TTL, Vault elimina automaticamente el usuario de la base de datos. Ni el cliente ni la aplicacion necesitan conocer las credenciales root.

### Sellos y desellos

Vault almacena sus secretos cifrados en disco, pero la clave de descifrado (el "unseal key") nunca se almacena en el servidor. Para que Vault arranque y pueda servir peticiones, un operador debe proporcionar las claves de deselado, que normalmente estan distribuidas entre multiples personas siguiendo un esquema de umbral (por ejemplo, 3 de 5 claves). Esto garantiza que ningun individuo aislado puede acceder a todos los secretos, incluyendo al propio administrador del servidor.

> Vault no es una solucion "plug and play". Requiere una infraestructura de soporte (cluster con almacenamiento persistente, politicas bien disenadas, integracion con los servicios consumidores) y un equipo que gestione su operacion. En entornos pequenos, la complejidad operativa puede superar los beneficios.

## Gestores de secretos en la nube

Los proveedores de nube ofrecen servicios gestionados de secretos que eliminan la necesidad de operar infraestructura propia. Son la opcion natural cuando la arquitectura reside completamente en un ecosistema de nube.

### AWS Secrets Manager

Secrets Manager de AWS permite almacenar, rotar y acceder a secretos de forma centralizada dentro del ecosistema de Amazon Web Services. Su principal ventaja es la integracion nativa con otros servicios AWS: Lambda puede acceder a secretos sin configuracion adicional, RDS puede rotar automaticamente las credenciales de bases de datos, y las politicas IAM controlan con granularidad quien puede leer o escribir cada secreto.

La rotacion automatica es uno de sus puntos fuertes. Secrets Manager ejecuta funciones Lambda personalizadas para rotar credenciales periodicamente, lo que lo convierte en una solucion completa de ciclo de vida de secretos sin intervencion manual. El coste por secreto almacenado por mes (aproximadamente 0.40 USD) y por peticion de la API (0.05 USD por 10,000) puede ser significativo a gran escala, por lo que conviene evaluar el volumen de secretos y la frecuencia de acceso antes de adoptarlo.

### Azure Key Vault

Azure Key Vault combina almacenamiento de secretos con gestion de claves de cifrado y certificados. Su modelo de acceso se basa en "vaults" que actuan como contenedores逻辑icos, cada uno con sus propias politicas de acceso definidas mediante Azure RBAC o politicas de acceso legacy.

Una caracteristica distintiva es la integracion con HSM (Hardware Security Modules) gestionados por Azure. Las claves criptograficas pueden almacenarse en modulos de hardware FIPS 140-2 nivel 2 o 3, lo que proporciona una garantia fisica de que las claves nunca salen del hardware. Esto es relevante para aplicaciones con requisitos de cumplimiento normativo estricto (banca, salud, administracion publica).

Key Vault permite tres niveles de secretos: claves (para operaciones criptograficas), secretos (para datos sensibles como contrasenas) y certificados (para la gestion del ciclo de vida de certificados TLS). Esta diferenciacion permite politicas de acceso y retencion distintas segun el tipo de dato.

### GCP Secret Manager

Secret Manager de Google Cloud Platform sigue un modelo similar pero con algunas diferencias de implementacion. Los secretos se organizan en proyectos de GCP y se accede a ellos mediante la API de Secret Manager o las bibliotecas oficiales. Cada version de un secreto se almacena de forma inmutable, lo que permite mantener historial de valores anteriores y desplegar retroactivamente una version especifica si es necesario.

Secret Manager integra automaticamente con Cloud IAM para el control de acceso, lo que permite definir politicas como "este servicio account solo puede leer esta version especifica de este secreto". La integracion con Cloud Build facilita el acceso a secretos durante los procesos de CI/CD sin necesidad de inyectarlos como variables de entorno.

## Sealed Secrets para Kubernetes

Sealed Secrets es un proyecto open source disenado especificamente para resolver el problema de gestionar secretos en repositorios de Kubernetes. La premisa es simple pero poderosa: permite cifrar secretos de Kubernetes de forma que puedan almacenarse de forma segura en control de versiones Git, algo que con los Secrets nativos de Kubernetes es imposible porque almacenan datos en base64 sin cifrar.

### Modelo de funcionamiento

Sealed Secrets opera mediante un controlador que se ejecuta dentro del cluster de Kubernetes. Para cifrar un secreto, se utiliza la herramienta `kubeseal` que se comunica con el controlador y obtiene la clave publica del cluster. El resultado es un SealedSecret (un recurso CRD de Kubernetes) que contiene el secreto cifrado y que solo puede ser descifrado por el controlador especifico de ese cluster.

El flujo es el siguiente: el desarrollador cifra el secreto con `kubeseal`, genera un YAML que se puede commitear de forma segura en el repositorio Git, y cuando ArgoCD, Flux u otra herramienta de GitOps aplica el manifesto al cluster, el controlador de Sealed Secrets lo descifra automaticamente y crea el recurso Secret nativo que las aplicaciones consumen.

### Propiedades de seguridad

El modelo tiene una propiedad fundamental: el secreto cifrado solo puede ser descifrado dentro del cluster destino, nunca fuera de el. Esto significa que incluso si alguien obtiene acceso al repositorio Git, solo vera datos cifrados inutilizables. La clave privada nunca abandona el cluster y se genera internamente por el controlador.

Sealed Secrets soporta scopes que restringen el secreto a un namespace y nombre especificos, lo que evita que un secreto cifrado para un namespace pueda ser reutilizado en otro. Tambien soporta rotation periodica mediante la renovacion de las claves de cifrado del controlador, que invalidan los valores anteriores y obligan a recifrar los secretos.

## SOPS: cifrado de archivos de configuracion

SOPS (Secrets OPerationS) es una herramienta de Mozilla disenada para cifrar archivos de configuracion que contienen secretos de forma que el resultado sea util y manejable. A diferencia de Vault o los gestores de nube, SOPS no proporciona un servicio centralizado: es una herramienta local que cifra y descifra archivos individuales.

### Que cifra y como

SOPS cifra los valores de un archivo de configuracion (YAML, JSON, INI, ENV) pero deja las claves en texto plano. Esto produce archivos donde las etiquetas de configuracion son legibles pero los valores sensibles aparecen cifrados con AES-256-CBC, AES-256-GCM o PGP. El resultado es un archivo que puede versionarse en Git, revisarse parcialmente (se pueden ver las claves pero no los valores) y aplicarse con herramientas existentes despues de descifrado.

### Claves de cifrado y distribucion

SOPS soporta multiples metodos de cifrado simultaneos: claves PGP, claves KMS de AWS o GCP, y claves de Azure Key Vault. Cuando se cifra un archivo, SOPS cifra el contenido una vez con una clave aleatoria (data key) y luego cifra la data key con cada una de las claves configuradas. Esto permite que multiples partes puedan descifrar el mismo archivo usando su propia clave, sin necesidad de compartir una contrasena comun.

La herramienta se integra con repositorios Git de forma que la clave de descifrado se obtiene automaticamente al hacer checkout del repositorio (por ejemplo, a traves de una clave GPG del desarrollador o de un role de IAM en CI/CD). Esto evita tener que pasar manualmente credenciales para acceder a los archivos cifrados.

## Comparativa y criterios de seleccion

Elegir entre estas herramientas depende de multiples factores que van mas alla de la funcionalidad tecnica pura.

| Criterio | Vault | Gestores nube | Sealed Secrets | SOPS |
|---|---|---|---|---|
| Complejidad operativa | Alta | Baja | Media | Baja |
| Modelo de despliegue | Autoservicio | Gestionado | Operador K8s | CLI local |
| Credenciales dinamicas | Si | Parcial (RDS) | No | No |
| Auditoria centralizada | Si | Si | Limitada | No |
| Coste | Infraestructura + licencia | Por secreto | Gratis (infra K8s) | Gratis |
| Multi-nube | Si | Limitado a un proveedor | Si | Si |
| GitOps nativo | No | No | Si | Si |

Vault es la opcion correcta cuando se necesita un control centralizado de secretos en entornos hibridos o multi-nube con requisitos estrictos de auditoria y credenciales dinamicas. Los gestores de nube son ideales cuando la infraestructura reside en un solo proveedor y se busca minimizar la carga operativa. Sealed Secrets encaja perfectamente en flujos GitOps de Kubernetes donde los secretos deben coexistir con el codigo de configuracion en Git. SOPS es la herramienta mas ligera y versatil para proyectos de cualquier tamano que necesiten cifrar archivos de configuracion sin infraestructura adicional.

> No existe una herramienta perfecta para todos los casos. Muchas organizaciones combinan multiples soluciones: Vault para secretos de servicios criticos, Sealed Secrets para GitOps en Kubernetes, y SOPS para archivos de configuracion cifrados en repositorios. La clave esta en definir claramente que tipo de secreto gestiona cada herramienta y documentar el proceso para el equipo.

---

